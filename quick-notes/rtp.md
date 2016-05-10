* RTP for audo and vid streaming, data protocol
* RTCP - Control protocol 
 * monitors transmission stats and QoS
 * Aids sync of streams

* RTP includes timestamps, seq numbers and payload format
* RTCP usually 5% of RTP bw
* RTP usually initiated by peers with signalling protocol (H323, SIP, XMPP), can use SDP to neg parameters
* RTP even ports, RTCP odd (i.e. next higher odd)
* RTP sessions ID' by 32 bit number SSRC (sync'd source)
* Five RTCP payload types
 * Sender report
 * Receiver report
 * Source Description
 * Goodbye
 * Application-defined packet
