SEVX - Simple EVent eXchange
============================

Send once, receive everywhere:
* Logs.
* Message queues.
* Workers.
* Stats.
* And so on.

protocol
--------

Client connects to server and sends one text line per request:

    {request_id} {action} {data}

Server responds:

    {request_id} ok 

Or server logs "error_id" with details, and responds:

    {request_id} error {error_id}

Server may respond multiple times for actions like "recv":

    {request_id} ok 
    {request_id} ok {data1}
    {request_id} ok {data2}
    ...

SEVX uses "dtid" for "request_id"-s, e.g. "20161231235959123456uN1q".  
See its pros and cons at https://github.com/denis-ryzhkov/uqidpy/blob/master/README.md#dtid

send
----

    from sevx import send, send_prefix
    send_prefix({"src": "service1"})

    send({"obj": "user", "act": "updated", "data": {"id": 42, "name": "Kate"}})

    > 20161231235959123456uN1q send {"src": "service1", "act": "updated", "obj": "user", "data": {"id": 42, "name": "Kate"}}
    < 20161231235959123456uN1q ok 

Server puts copies of this event to zero or more queues that were subscribed to this event.

recv
----

Worker may subscribe to zero or more keys:

    from sevx import on

    @on({"obj": "user", "act": "updated"})
    def on_event(event):
        event.id                    # 20161231235959123456uN1q
        event.dt                    # datetime(2016, 12, 31, 23, 59, 59, 123456)
        event.json                  # '{"src": "service1", "act": "updated", "obj": "user", "data": {"id": 42, "name": "Kate"}}'
        event.content.data.name     # 'Kate'

    > 20161231235959123456uN22 recv {"pattern": {"obj": "user", "act": "updated"}, "queue": "opt.service2.events.test.on_event.act=updated.obj=user"}
    < 20161231235959123456uN22 ok
    < 20161231235959123456uN22 ok 20161231235959123456uN1q {"src": "service1", "act": "updated", "obj": "user", "data": {"id": 42, "name": "Kate"}}

Default queue name is composed from full path to func + sorted pattern,  
e.g. "opt.service2.events.test.on_event.act=updated.obj=user".  
So workers of the same type share the same queue and the same event will not be processed twice.

    @on(pattern,
        delete_queue_when_unused=
            True,   # default
            False,
            5.42,   # seconds
        manual_ack=True,
        wait=True,
    )
    def on_event(event):
        event.retry     # 0, 1, 2, etc

    > 20161231235959123456uN33 recv {"manual_ack": true, "delete_queue_when_unused": 5.42, "queue": "opt.service2.events.test.on_event.act=updated.obj=user", "pattern": {"obj": "user", "act": "updated"}}
    < 20161231235959123456uN33 ok 
    < 20161231235959123456uN33 ok 20161231235959123456uN1q {"retry": 1, "src": "service1", "act": "updated", "obj": "user", "data": {"id": 42, "name": "Kate"}}

        # Acknowledge that this (or all) events were processed by this receiver.
        event.ack()             # 20161231235959123456uN44 ack {"receiver_id": "20161231235959123456uN33", "event_id": "20161231235959123456uN1q"}
        event.ack_all()         # 20161231235959123456uN44 ack {"receiver_id": "20161231235959123456uN33", "all": true}

        # Reject this (or all) events - to return them to the queue with incremented "retry" counter.
        event.reject()          # 20161231235959123456uN55 reject {"receiver_id": "20161231235959123456uN33", "event_id": "20161231235959123456uN1q"}
        event.reject_all()      # 20161231235959123456uN55 reject {"receiver_id": "20161231235959123456uN33", "all": true}

    on_event.delete_queue()     # 20161231235959123456uN66 delete_queue {"queue": "opt.service2.events.test.on_event.act=updated.obj=user"}
    # Server will not copy events to this queue any more.

    on_event.delete()           # 20161231235959123456uN77 delete_receiver {"receiver_id": "20161231235959123456uN33"}
    # Server will not send events to this receiver any more.

Receiving in manual-ack mode spends more resources, so avoid it when possible for 2x better performance.

wait
----

* Client may "wait=True" for "ok" or "error" - safe, slow.
* Or client may proceed async - fast, default.
* "wait" is usefull mostly in tests, to ensure that "on_event()" is ready to receive events.
* Please do not use "send(wait=True)" in production - it is very slow.
* But "@on(wait=True)" is OK, because client waits only first empty "ok" response.
* "wait" option is not sent to server, because it is completely client's feature.

disconnect
----------

* Server deletes all receivers of disconnected client.
* If receiver with manual-ack is deleted, all not-acked events are automatically rejected by server - returned to the queue.
* Client restarts all not-deleted receivers on autoreconnect to server.

internal
--------

    import sevx

    sevx._ping('service1')  # Is used by sevx client for keep-alive.
    > 20161231235959123456uN00 _ping {"data": "service1"}
    < 20161231235959123456uN00 ok {"data": "service1"}

    sevx._eval('len(state.queues)')  # Backdoor to get any stats.
    > 20161231235959123456uN00 _eval {"code": "len(state.queues)"}
    < 20161231235959123456uN00 ok {"result": 42}

end
---

sevx version 0.1.0  
Copyright (C) 2016 by Denis Ryzhkov <denisr@denisr.com>  
MIT License, see http://opensource.org/licenses/MIT
