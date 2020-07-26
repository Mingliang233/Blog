### 锁

### innoDB行锁



1.记录锁record lock

2.间隙锁gap lock

###### 只存在RR中

3.临键锁next-key lock

next-key lock = gap lock + record lock

select ... in share mode

select ... for update

RR RC

MVCC



Read Uncommited

Serilizeable



意向锁

排他锁

共享锁