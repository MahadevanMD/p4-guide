# Compiling

See README-using-bmv2.txt for some things that are common across
different P4 programs executed using bmv2.

To compile the P4_16 version of the code:

    p4c-bm2-ss checksum-ipv4-with-options.p4 -o checksum-ipv4-with-options.json

This program currently only compiles without errors using a version of
p4c after this 2017-Aug-23 commit:

    https://github.com/p4lang/p4c/commit/f09a95dcda68ca18a1528be6a63a16f2b8074d46

but before that commit was reverted on 2017-Aug-28.  I have checked in
a file checksum-ipv4-with-options.json generated with the p4c-bm2-ss
command above, using a version of p4c-bm2-ss compiled from the p4c
repository, just before the 2017-Aug-28 commit that reverted the
needed changes.

# Running

To run the behavioral model with 8 ports numbered 0 through 7:

    sudo simple_switch --log-console -i 0@veth2 -i 1@veth4 -i 2@veth6 -i 3@veth8 -i 4@veth10 -i 5@veth12 -i 6@veth14 -i 7@veth16 checksum-ipv4-with-options.json


No table entries need to be added for the default action 'foo' to be
run on all packets.


----------------------------------------------------------------------
scapy session for sending packets
----------------------------------------------------------------------
We must run scapy as root for it to have permission to send packets on
veth interfaces.

I found a working example of using Scapy to generate IPv4 headers with
IP options here: http://allievi.sssup.it/techblog/archives/631

    sudo scapy

    pkt1=Ether() / IP(dst='10.1.0.1') / TCP(sport=5793, dport=80)
    pkt2=Ether() / IP(dst='10.2.3.4', options=IPOption('\x83\x03\x10')) / TCP(sport=5501, dport=80)

    # Send packet at layer2, specifying interface
    sendp(pkt1, iface="veth2")
    sendp(pkt2, iface="veth2")

The file `packets-in-port0.pcap` in this directory contains a capture
of the input packets generated by the above Scapy function calls, as
sent to simple_switch.

The file `packets-out-port0.pcap` contains a capture of the output
packets produced by the checksum-ipv4-with-options.p4 program, when
given the above packets as input.

    % cp packets-in-port0.pcap 0_in.pcap

    # This command will read the pcap file 0_in.pcap for packets to
    # send into port 0 of simple_switch, and write output packets
    # transmitted on port 0 to file 0_out.pcap.
    
    % simple_switch -i 0@0 --use-files 0 checksum-ipv4-with-options.json

    [ Use Ctrl-C to kill simple_switch process after a second or two
      of inactivity ]

    % ls -l 0_out.pcap
    -rw-r--r-- 1 jafinger jafinger 168 Sep  5 02:31 0_out.pcap

    # Use editcap to change the link type of the pcap file from 'None'
    # to Ethernet, and the snaplen from 0 to 262144.  The resulting
    # output file 0_out_ether.pcap is more easily usable by programs
    # that read and process pcap files.
    
    % editcap -F pcap -T ether 0_out.pcap 0_out_ether.pcap

    # Use Scapy to verify that the packets out in 0_out_ether.pcap are
    # the same as those in packets-out-port0.pcap
    
    % scapy
    Welcome to Scapy (2.3.3)
    >>> p1=rdpcap('packets-out-port0.pcap')
    >>> p2=rdpcap('0_out_ether.pcap')
    >>> len(p1)
    2
    >>> len(p2)
    2
    >>> str(p1[0])==str(p2[0])
    True
    >>> str(p1[1])==str(p2[1])
    True

----------------------------------------
