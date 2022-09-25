---
title: ORACLE sql 分析
date: 2022-08-31 14:11:21
tags:
- oracle
categories:
- DB
---


EXPLAIN PLAN FOR select * from A_AGENT a where a.ID='AG20181121000000000011525';

--SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('PLAN_TABLE'));

select * from table(dbms_xplan.display);