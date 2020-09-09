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
    - 현재 WAIT command를 호출한 클라이언트가 있다면 → Grouping 해서 한번에 처리
    - 현재 WAIT command를 호출한 클라이언트가 없다면 → REPLCONF GETACK command를 Replica로 propagate함
        - Replica는 REPLCONF GETACK을 받자마자 현재까지 처리한 Replication offset을 master로 전송(ACK)
        - WAIT command parameter로 전달된 replication 수만큼 ACK을 받자마자 blocking을 풀고 client로 응답 전송

## Client 사용 예시
- 3개 이상의 Replica가 살아있을 때만 payment_id를 save하는 예제
        - wait가 3이상 return할 시에만 confirmed를 저장함
```bash
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
