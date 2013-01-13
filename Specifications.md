openbts-handover
================

Roaming
-------

Roadming in OpenBTS is realized by mapping GSM Location Updating Request to SIP REGISTER (with TMSI assignement) [code](https://github.com/ttsou/openbts-p2.8/blob/umtrx/Control/MobilityManagement.cpp#L136), and GSM IMSI Detach Indication (phone shutting off) to SIP (un)REGISTER [code](https://github.com/ttsou/openbts-p2.8/blob/umtrx/Control/MobilityManagement.cpp#L87). The registrar is used to map IMSI+number to username+ip+port [code](https://github.com/ttsou/openbts-p2.8/blob/umtrx/FreeswitchConfig/dialplan/openbts-dialplan.xml).

Handover
--------
Handover is not implemented in the public version of OpenBTS. The wikipedia article on Handover has some [links](https://en.wikipedia.org/wiki/Handover#External_links) that describe the Handover process. For now I'm going to focus on inter-cell handover, since intra-cell handover would most probably be implemented inside OpenBTS and be transparent on the SIP side.

Call Flow
---------

Compare this call flow to the document [Intra-MSC GSM Handover Call Flow](http://www.eventhelix.com/RealtimeMantra/Telecom/GSM_Handover_Call_Flow.pdf) referenced on [wikipedia](https://en.wikipedia.org/wiki/Handover#External_links).

    Mobile        OpenBTS(target)     OpenBTS(origin)         SIP

    ----  RR Measurement Report ------->
                                          ----  REFER ------>  Referred-By: origin (insecure)
                                          <---- 202 ---------
                                          (<-- NOTIFY ------)
                                          (--- 200 -------->)
                        <------ INVITE ----------------------
                        (------- 180 ---------------------->)
                    FIXME: Here we need some kind of way to send the
                    Handover info back to the origin cell.
                        -- INVITE  ---->
                        <- 404 ---------
    <----- RR Handover Command ---------
    -- RR Hand. Accept ->
                        -------- 200 ----------------------->
                        <------- ACK ------------------------
                                          (<-- NOTIFY ------)
                                          (--- 200 -------->)
    <-- RR Phys.Info.  --
    --- RR SABM -------->
    <-- RR UA -----------
    --- RR Hand. Cplte ->
                        -------- REGISTER ------------------>
                        <------- 200 ------------------------

                                          <---- BYE ---------
                                          ----- 200 -------->
                                          --- unREGISTER --->
                                          <---- 200 ---------

Cell losing the mobile (origin cell)
------------------------------------

From the point of view of the cell losing the mobile, the handling of the RR Measurement Report is what triggers the handover decision. The messages exchanged normally with a VLR are:

* BSC -> VLR: BSSMAP Handover Required (trigerred by detection of condition requiring a handover)
* BSC <- VLR: BSSMAP Handover Command (triggers the RR Handover Command to the mobile)
* BSC <- VLR: BSSMAP Clear Command (once the handover has been confirmed)
* BSC -> VLR: BSSMAP Clear Complete

Clearly the last two messages of the call flow are mapped to SIP BYE and 200 OK. The BYE message should not be forwarded to the mobile, so the state of the call should be modified at the start of the handover to prevent it.

Since the BSSMAP Handover Required message must indicate the Target Cell(s), I propose it be mapped to a SIP REFER message. (However the Refer-To header of the REFER method does not allow for multiple targets to be referenced, so it might be necessary to use a forking proxy and generic domain names such as "cell-123-neighbors". In any case this processing is left to the SIP side, and OpenBTS only needs to send the REFER message.)

Alternatively these messages could be mapped to an un-REGISTER message, but the REGISTER message does not belong to the SIP dialog (=~ call) so it would make the mapping more difficult. Other options (INFO message, reINVITE) seem more complicated to implement.

As indicated, the domain name of the Refer-To SIP URI is used to indicate the server(s) that will be contacted to process the handover. **The following is incorrect, 202 is received even before INVITE is sent out, so another mechanism must be found.** The 202 Accepted response will contain (probably in some X- header, or maybe as a parameter in the Contact header?) the information required for OpenBTS to build the RR Handover (handover reference, TCH on the target OpenBTS).

Lastly, OpenBTS will send an unREGISTER message at the end of the call (when it receives the 200 OK response to the BYE).

Cell gaining the mobile (target cell)
-------------------------------------

From the point of view of the target cell, the handover will start with an INVITE message triggered on the SIP side by the REFER message of the losing cell. The INVITE must contain some indication that it is related to a handover procedure.

The reception of the INVITE will lead to the allocation of the TCH, handover reference, etc. which will be sent to the SIP side inside a 180 SIP message.  When the SIP side receives the 180 OK message, it can then build the 202 Accepted response to the REFER message it received from the losing cell. In OpenBTS, the INVITE will not be forwarded to the mobile; OpenBTS has to wait for the mobile to signal.

Next the target cell should receive a RR Handover Accept message from the mobile. This is mapped to 200 OK SIP message in order to switch the call to the new audio path.

Then the target cell sends out RR Physical Information (this sequence might already be coded in OpenBTS), mobile responds with RR SABM, target sends RR UA and mobile replies with RR Handover Complete.

There isn't a clear mapping for the BSSMAP Handover Complete message; most probably at that point the SIP side will already have sent the BYE message to the origin cell. However a REGISTER message will have to be sent out by the target cell.
