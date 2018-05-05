---
layout:     post
title:      stateful openflow
subtitle:
date:       2017-4-2
author:     dodoYuan
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - openflow
---

#####整理openstate原理以及具体应用

```
LOG = logging.getLogger('app.openstate.portknock')

""" Last port is the one to be opened after knocking all the others """
port_list = [10, 11, 12, 13, 22]
final_port = port_list[-1]
second_last_port =  port_list[-2]

LOG.info("Port knock sequence is %s" % port_list[0:-1])
LOG.info("Final port to open is %s" % port_list[-1])

class OpenStatePortKnocking(app_manager.RyuApp):

    def __init__(self, *args, **kwargs):
		super(OpenStatePortKnocking, self).__init__(*args, **kwargs)

	@set_ev_cls(ofp_event.EventOFPSwitchFeatures, CONFIG_DISPATCHER)
	def switch_features_handler(self, ev):
		msg = ev.msg
		datapath = msg.datapath

		LOG.info("Configuring switch %d..." % datapath.id)

		""" Set table 0 as stateful """
		req = osparser.OFPExpMsgConfigureStatefulTable(datapath=datapath,
													table_id=0,
													stateful=1)
		datapath.send_msg(req)

		""" Set lookup extractor = {ip_src} """
		req = osparser.OFPExpMsgKeyExtract(datapath=datapath,
										command=osproto.OFPSC_EXP_SET_L_EXTRACTOR,
										fields=[ofproto.OXM_OF_IPV4_SRC],
										table_id=0)
		datapath.send_msg(req)

		""" Set update extractor = {ip_src} (same as lookup) """
		req = osparser.OFPExpMsgKeyExtract(datapath=datapath,
									command=osproto.OFPSC_EXP_SET_U_EXTRACTOR,
									fields=[ofproto.OXM_OF_IPV4_SRC],
									table_id=0)
		datapath.send_msg(req)

		""" ARP packets flooding """
		match = ofparser.OFPMatch(eth_type=0x0806)
		actions = [ofparser.OFPActionOutput(ofproto.OFPP_FLOOD)]
		self.add_flow(datapath=datapath, table_id=0, priority=100,
						match=match, actions=actions)

		# 这个是可以理解的，下发流表
		""" Flow entries for port knocking """
		for i in range(len(port_list)):
			match = ofparser.OFPMatch(eth_type=0x0800, ip_proto=17,
										state=i, udp_dst=port_list[i])

			if port_list[i] != final_port and port_list[i] != second_last_port:
				# If state not OPEN, set state and drop (implicit)
				actions = [osparser.OFPExpActionSetState(state=i+1, table_id=0, idle_timeout=5)]
			elif port_list[i] == second_last_port:
				# In the transaction to the OPEN state, the timeout is set to 10 sec
				actions = [osparser.OFPExpActionSetState(state=i+1, table_id=0, idle_timeout=10)]
			else:
				actions = [ofparser.OFPActionOutput(2)]
			self.add_flow(datapath=datapath, table_id=0, priority=10,
							match=match, actions=actions)

		""" Get back to DEFAULT if wrong knock (UDP match, lowest priority) """
		match = ofparser.OFPMatch(eth_type=0x0800, ip_proto=17)
		actions = [osparser.OFPExpActionSetState(state=0, table_id=0)]
		self.add_flow(datapath=datapath, table_id=0, priority=0,
						match=match, actions=actions)

		""" Test port 1300, always forward on port 2 """
		match = ofparser.OFPMatch(eth_type=0x0800, ip_proto=17, udp_dst=1300)
		actions = [ofparser.OFPActionOutput(2)]
		self.add_flow(datapath=datapath, table_id=0, priority=10,
						match=match, actions=actions)


	def add_flow(self, datapath, table_id, priority, match, actions):
		inst = [ofparser.OFPInstructionActions(
				ofproto.OFPIT_APPLY_ACTIONS, actions)]
		mod = ofparser.OFPFlowMod(datapath=datapath, table_id=table_id,
								priority=priority, match=match, instructions=inst)
		datapath.send_msg(mod)
```
openstate基本思想就是控制器下放一部分功能，交换机不再是简单的dumb，而是保留一些简单的wise。
论文中以端口锁定为例，提出了米粒型状态机在交换机内部的应用从而可以大大减少交换机和控制器之间的交互，减缓了控制器的性能瓶颈。
传统SDN架构，对一些安全应用如端口锁定，只有一定顺序的端口请求才会开放目标端口，如请求22端口进行数据传输，但设置的端口请求列表是[10,12,13,14,22]，就是说只有在之前目标主机依次收到这些端口请求后才会开放22号端口。这个策略如果需要由控制器来完成，将会十分繁琐，每次来一个数据包请求控制器，控制器根据历史端口生成处理策略。
论文中指出，SDN的最大优势就是全局能力，而这样的操作完全是本交换机自己的事情，交给控制器处理并没有获得控制器的优势，反而增加了控制器的处理负担。因此提出了交换机内部可以维持一个状态机。
![image.png](http://upload-images.jianshu.io/upload_images/3635313-71e1fdb0706c4884.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

要实现上述
交换机维持两张表，状态表和流表，流表在原来的基础上进行了扩充，增加了state的查询。具体如下：
![image.png](http://upload-images.jianshu.io/upload_images/3635313-904383edba136ecb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
知识补充：lookup 和 update
lookup和update是交换机实现的功能，由lookup extractor和update extractor两个功能模块完成，在交换机中实现。两个功能模块作用就是在数据包中取出唯一标识符，可以由控制器自定义。当数据包进入交换机时，调用lookup extractor，比如以源IP为标识符，则为这条数据流生成了以IP地址为标识的唯一标识符。交换机根据这条标识去state table中查找对应的状态，并以该状态和其他匹配字段去查询流表，最终完成了整个功能实现。

```
class OpenStateMacLearning(app_manager.RyuApp):
    def __init__(self, *args, **kwargs):
		super(OpenStateMacLearning, self).__init__(*args, **kwargs)

	def add_flow(self, datapath, table_id, priority, match, actions):
		if len(actions) > 0:
			inst = [ofparser.OFPInstructionActions(
					ofproto.OFPIT_APPLY_ACTIONS, actions)]
		else:
			inst = []
		mod = ofparser.OFPFlowMod(datapath=datapath, table_id=table_id,
								priority=priority, match=match, instructions=inst)
		datapath.send_msg(mod)

	@set_ev_cls(ofp_event.EventOFPSwitchFeatures, CONFIG_DISPATCHER)
	def switch_features_handler(self, event):

		""" Switche sent his features, check if OpenState supported """
		msg = event.msg
		datapath = msg.datapath

		LOG.info("Configuring switch %d..." % datapath.id)

		""" Set table 0 as stateful """
		req = osparser.OFPExpMsgConfigureStatefulTable(
				datapath=datapath,
				table_id=0,
				stateful=1)
		datapath.send_msg(req)

		""" Set lookup extractor = {eth_dst} """
		req = osparser.OFPExpMsgKeyExtract(datapath=datapath,
				command=osproto.OFPSC_EXP_SET_L_EXTRACTOR,
				fields=[ofproto.OXM_OF_ETH_DST],
				table_id=0)
		datapath.send_msg(req)

		""" Set update extractor = {eth_src}  """
		req = osparser.OFPExpMsgKeyExtract(datapath=datapath,
				command=osproto.OFPSC_EXP_SET_U_EXTRACTOR,
				fields=[ofproto.OXM_OF_ETH_SRC],
				table_id=0)
		datapath.send_msg(req)

		# for each input port, for each state
		for i in range(1, N+1):
			for s in range(N+1):
				match = ofparser.OFPMatch(in_port=i, state=s)
				if s == 0:
					out_port = ofproto.OFPP_FLOOD
				else:
					out_port = s
				actions = [osparser.OFPExpActionSetState(state=i, table_id=0, hard_timeout=10),
							ofparser.OFPActionOutput(out_port)]
				self.add_flow(datapath=datapath, table_id=0, priority=0,
								match=match, actions=actions)
```
上面是用来进行端口学习的，乍看还是有点难懂。他最大的好处就是不用和控制器进行交互，提前预下发流表，然后在交换机内部根据状态信息进行学习。主要思想如下：
第一步：给所有的（假设n个）端口下发n+1个带状态的流表。如4口交换机则端口1就有生成5个匹配流表，匹配项为入端口号和state值，action为outport，出端口值就是非0的state值，state为0表示泛洪。
但交换机又是如何学习的呢，一条数据流进入，首先通过lookup查询当前流的状态，当没有学习到目的地址时，state值还是为0，泛洪。同时，因为update以源MAC地址更新，匹配到流表后会取出源MAC地址，更新state为进端口号，`osparser.OFPExpActionSetState(state=i, table_id=0, hard_timeout=10)`
所以下次当有该目的MAC地址来世，查询到的state也就是出端口号。设计的有点反常识，但还是证明stateful 数据平面确实是挺强大的，可以在交换机内部完成一些简单的操作。
