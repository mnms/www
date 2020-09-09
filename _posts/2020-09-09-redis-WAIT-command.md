---
layout: blog
title: REDIS의 WAIT Command
author: Dooyoung Hwang
published: true
category: tech-review
excerpt: |
  WAIT Command는 REDIS master와 replica 사이의 asynchronous replication 설정 시 발생할 수 있는 데이터 일관성 문제를 보완하기 위한 Command입니다. REDIS 창시자의 블로그의 "WAIT: synchronous replication for Redis" (http://antirez.com/news/66)를 요약, 정리하였습니다.
---

## Synchronous vs Asynchronous Replication
- Synchronous : Write 시 모든 replication으로부터 ACK을 받고 Client에게 OK전송 - latency 문제
- Asynchronous : Write 시 replication으로부터 ACK을 받기 전에 Client에게 OK전송 - Redis에서 default로 채택하는 방식
- Asynchronous 방식 채택 시 master와 replica간의 Consistency 문제는 ? WAIT Command를 통해 해결함

## Example
```bash
redis 127.0.0.1:9999> set foo bar
OK
redis 127.0.0.1:9999> incr mycounter
(integer) 1

# 5개의 Replica로 Write가 Propagate될 때까지 MAX 100ms동안 기다림
# 현재까지 Write command를 sync완료한 replica의 갯수가 return
redis 127.0.0.1:9999> wait 5 100
(integer) 7
```

## 동작 방식
- global offset : replica로 Command를 propagate할 때마다 증가되는 offset
- replication offset : 모든 replica는 현재까지 처리완료한 offset을 저장하고 있음
    - 매초마다 replica는 replication offset을 master로 전송(ACK)
- WAIT command가 호출될 시
    - REPLCONF GETACK command를 Replica로 propagate함
        - Replica는 REPLCONF GETACK을 받자마자 현재까지 처리한 Replication offset을 master로 전송(ACK)
        - WAIT command parameter로 전달된 replication 수만큼 ACK을 받자마자 blocking을 풀고 client로 응답 전송

## Client 사용 예시
- 3개 이상의 Replica가 살아있을 때만 payment_id를 save하는 예제
        - wait가 3이상 return할 시에만 confirmed를 저장함
```python
def save_payment(payment_id)
        redis.rpush(payment_id,”in progress”) # Return false on exception
        if redis.wait(3,1000) >= 3 then
            redis.rpush(payment_id,”confirmed”) # Return false on exception
            if redis.wait(3,1000) >= 3 then
                return true
            else
                redis.rpush(payment_id,”cancelled”)
                return false
            end
        else
            return false
    end
```

## REDIS Source code
- redis 5.0 부분 소스 코드를 발췌
- waitCommand 구현 부분 (replication.c)
```c
void waitCommand(client *c) {
    mstime_t timeout;
    long numreplicas, ackreplicas;
    // offset 변수는 command를 수신한 시점의 global offset이다.
    // 결국 replica들의 offset이 이 offset변수만큼 sync되는 걸 기다린다고 보면 된다.
    long long offset = c->woff;
    ...
    // global offset만큼 sync된 replica 수가 client에서 질의한 replica수보다 크다면 blocking없이 바로 return한다.
    ackreplicas = replicationCountAcksByOffset(c->woff);
    if (ackreplicas >= numreplicas || c->flags & CLIENT_MULTI) {
        addReplyLongLong(c,ackreplicas);
        return;
    }
    // Blocking된 client list에 추가하고 client block후 replica로부터의 ACK을 기다린다.
    c->bpop.timeout = timeout;
    c->bpop.reploffset = offset;
    c->bpop.numreplicas = numreplicas;
    listAddNodeTail(server.clients_waiting_acks,c);
    blockClient(c,BLOCKED_WAIT);
    // REPLCONF GETACK Command를 replica로 보낸다.
    // (실제로는 replicationRequestAckFromSlaves에서는 바로 REPLCONF GETACK을 보내진 않고 flag설정 후 다음 event loop에서 전송한다. 이는 WAIT요청이 여러 클라이언트로 부터 동시에 이루어져도 한 event loop내에서는 한번만 REPLCONF GETACK을 보내기 위한 처리로 보여진다.)
    replicationRequestAckFromSlaves();
}

```
- Replica의 REPLCONF GETACK 처리 부분 (replication.c)
```c
void replconfCommand(client *c) {
    ...
     else if (!strcasecmp(c->argv[j]->ptr,"getack")) {
            // REPLCONF GETACK을 수신한 replica는 즉시 현재 sync offset을 master로 전송한다.
            if (server.masterhost && server.master) replicationSendAck();
            return;
        }
    ...    
}
```
- Client를 unblock하는 부분 (server.c)
```c
// wait Command의 다음 event loop 시작점에서 호출됨
void beforeSleep(struct aeEventLoop *eventLoop) {
    ...
    // waitCommand에서 설정한 get_ack_from_slaves flag가 ON일 경우 REPLCONF GETACK을 replica들에게 전송함
    if (server.get_ack_from_slaves) {
        robj *argv[3];

        argv[0] = createStringObject("REPLCONF",8);
        argv[1] = createStringObject("GETACK",6);
        argv[2] = createStringObject("*",1); /* Not used argument. */
        replicationFeedSlaves(server.slaves, server.slaveseldb, argv, 3);
        decrRefCount(argv[0]);
        decrRefCount(argv[1]);
        decrRefCount(argv[2]);
        server.get_ack_from_slaves = 0;
    }
     // replica들의 replication offset을 기준으로 WAIT 상태의 client들의 unblock여부를 판단함
    if (listLength(server.clients_waiting_acks))
        processClientsWaitingReplicas();
    ...
}
```
