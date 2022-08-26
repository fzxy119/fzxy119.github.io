---
title: vmtoll
date: 2022-08-26 17:09:06
tags:
- arthas
categories:
- 技术
---


7490376  Approved
sc -d org.springframework.context.ApplicationContext
--线上执行代码

vmtool --action getInstances -c  6981e5e3 --className org.springframework.context.ApplicationContext --express 'instances[1].getBean("snReturnService").approveReject("5020059")'

vmtool --action getInstances -c 2f9cd665 --className org.springframework.context.ApplicationContext --express 'instances[1].getBean("orderAdjustService").approveFinishOrderAdjust("4922565","reject_end")'

vmtool --action getInstances -c 5950a341 --className org.springframework.context.ApplicationContext --express 'instances[1].getBean("activityService").single("snreturn_single_reject","4735433")'

vmtool --action getInstances -c 5950a341 --className org.springframework.context.ApplicationContext --express 'instances[1].getBean("kfkCallBackEnterService").dealMsg("KM210317001134012","3","解冻完成")'

vmtool --action getInstances -c 5950a341 --className org.springframework.context.ApplicationContext --express 'instances[1].getBean("agentContractService").queryRedisAgContractFreeze()'

vmtool --action getInstances -c 2f9cd665 --className org.springframework.context.ApplicationContext --express 'instances[1].getBean("dataChangeActivityService").compressColInfoDataChangeActivity("4877809","Approved")'

vmtool --action getInstances -c   2af28f0e --className org.springframework.context.ApplicationContext --express '#arr = new java.util.ArrayList<java.lang.String>(), #arr.add("5020059"), instances[1].getBean("activityService").deleteInstance(#arr)'

