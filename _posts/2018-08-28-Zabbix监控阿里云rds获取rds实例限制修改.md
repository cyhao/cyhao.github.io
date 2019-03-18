---
layout:     post
title:      Zabbix监控阿里云rds获取rds实例限制修改
subtitle:   Zabbix监控阿里云rds获取rds实例限制修改
date:       2018-08-28
author:     cyhao
catalog: true
tags:
    - zabbix
    - 阿里云
    - rds
---

>下载的discovery_rds.py默认是最多获取30条rds实例

修改discovery_rds.py脚本可以实现多实例的发现，修改内容如下

原先：

     def GetRdsList():
        RdsRequest = DescribeDBInstancesRequest.DescribeDBInstancesRequest()
        RdsRequest.set_accept_format('json')
        #RdsInfo = clt.do_action(RdsRequest)
        RdsInfo = clt.do_action_with_exception(RdsRequest)
        for RdsInfoJson in (json.loads(RdsInfo))['Items']['DBInstance']:


修改后：

      def GetRdsList():
        RdsRequest = DescribeDBInstancesRequest.DescribeDBInstancesRequest()
        RdsRequest.set_accept_format('json')
        request.set_PageSize(100)   #每页实例条数
        request.set_PageNumber(1)    #第几页
        #RdsInfo = clt.do_action(RdsRequest)
        RdsInfo = clt.do_action_with_exception(RdsRequest)
        for RdsInfoJson in (json.loads(RdsInfo))['Items']['DBInstance']:

