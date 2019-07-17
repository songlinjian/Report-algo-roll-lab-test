## Introduction
 
In this April we called participation of second algorithm rollover in a controlled lab environment. We try rolled the algorithm rollover in root level with four approaches and record the monitoring data every 10 minutes. The results help us better understand the protocol as well as the gap between protocol and implementation. In this post we skip a full introduction of algorithm rollover background and the testing proposal of the Second algorithm rollover test. Because that information was well documented in previous posts : [Call for participation](https://yeti-dns.org/yeti/blog/2019/04/02/call-for-participation-algorithm-roll.html) and [Algorithm test and monitoring page](https://yeti-dns.org/alg-roll-test.html). Instead, this post only demonstrates the interesting results and findings which give us a second thought on algorithm rollover for root.

Note that in this test we use BIND9.11.5-P1, UNBOUND1.8.3, PowerDNS recursor 4.2.0 alpha1 as our testing resolvers.

## Experience and Findings

### Use of dnssec-signzone

We use BIND dnssec-signzone tool to sign the zone, generate NSEC and RRSIG records and produces a signed version of the zone. In schedule of the test (described in [test and monitoring page](https://yeti-dns.org/alg-roll-test.html)), it is important to manipulate the keys to sign the zone and DNSKEY Records. This is achieved by manipulate the key repository of dnsse-signzone with -K option). 

For example, in the test we use dnssec-signzone this way :

>$ dnssec-signzone -K ${ROOT_KEY} -P -o . -O full -S -x ${ZONE_DATA}/root.zone 

in which, -K option specifies a directory to search for DNSSEC keys (private keys in the directory are used to sign). And -x option is used to only sign the DNSKEY RRset with key-signing keys. Note that we use -P to disable post sign verification tests in Case 1 which will be introduce later. ([More information of options of dnssec-signzone](https://bind.isc.org/doc/arm/9.11/man.dnssec-signzone.html)).


<!---P option are used so that we can roll the algorithm without a non revoked self signed KSK key as we planed, because the old RSA KSK is going to be revoked and deleted in the time line. Note that -->

Everything seems perfect using dnssec-signzone to roll the algorithm by placing different keys in ROOT_KEY directory. However, it was found that when old RSA KSK is revoked or deleted, the RSA ZSK will be used to sign both the zone and DNSKEY records, even when -x option is set. -x option means "only sign the DNSKEY RRset with key-signing keys".

There is a guess that smart signing (-S option) of BIND will ignore key flag when there is only a private key available for one algorithm. As a result BIND signs DNSKEY records with both New ECDSA KSK and old RSA ZSK. We corrected this with special script manually  by stripping the DNSKEY RRSIG by RSA ZSK when the algorithm was really rolled from RSA to ECDSA. Operators who are going to implement algorithm rollover using dnssec-signzoe should notice and take care of it.

### Double-DS for Algorithm Rollover in Case 1

Both [RFC4035](https://tools.ietf.org/html/rfc4035#section-2.1) and [RFC6781](https://tools.ietf.org/html/rfc6781#section-4.1.2) specify that "There MUST be an RRSIG for each RRset using at least one DNSKEY of each algorithm in the zone apex DNSKEY RRset." It is inferred that it is the reason of excluding the Double-DS approach for algorithm roll because "when adding a new algorithm, the signatures should be added first"

Case 1 is designed to test against this specification in which the new algorithm (new ECDSA KSK) is re-published without corresponding RRSIG. Firstly we use dnssec-signzone without -P option, and the post sign verification tests failed. With -P option, a signed zone was generated and published. 

On the resolver side, it is seems that resolver have no sense of that specification at all (requirement). The introduction of pre-published ECDSA KSK transitioned the state of resolver from Start to AddPend and start the Add Hold-Down Time ([RFC5011 State table](https://tools.ietf.org/html/rfc5011#section-4)). And the state transitioned to Valid state after the timer expired. In addition no validation failure was reported during the Algorithm rollover in case 1. 

It is known that Double-DS has the benefit of generating smaller size of DNS response. In the context of RFC5011, it requires no exchange with parent in root level. Exchanging with parent is viewed as the major drawback of Double-DS approach. So the question comes: is it worth of a second thought on Double-DS for algorithm rollover in future ? Any risk ?

### Difference behaviors between BIND and Unbound NO.1 (To be confirmed)

It was reported in our [first algorithm rollover test](https://yeti-dns.org/yeti/blog/2019/04/02/call-for-participation-algorithm-roll.html) that BIND and Unbound responsed differently when they had validation failures. 

In the first test, an misconfiguration of old RSA KSK with an earlier inactive time which results no active signing key in the middle of the rollover and causes validation failure. We reset the old RSA KSK to be active but it still had impact on resolvers. As a response to this failure, it was observed BIND resolver restarts the Add Hold-Down Timer of the new key/algorithm for another 30 days when the old RSA KSK became active.

In the contrast, Unbound's timer did not stoped and trusted the KSK/Algorithm after the timer expired. It is inferred that BIND treats the failure as KeyRem event, but UNBOUND treat it as a non-rfc5011 event like network failure. <!-- UNBOUND stops the timer and continues it after failure resolved. -->

### Difference behaviors between BIND and Unbound NO.2

[RFC5011](https://tools.ietf.org/html/rfc5011#section-4) specifies that a resolver MUST query that trust point for an automatic update of keys from a particular trust point. e.g., do a lookup for the DNSKEY RRSet and related RRSIG records. This operation is called Active Refresh. 

According to the formula suggest in RFC5011, queryInterval = MAX(1 hr, MIN (15 days, 1/2*OrigTTL, 1/2*RRSigExpirationInterval)). Because the TTL of DNSKEY RRSET is set to 600 second to remove the impact of cache in this test. So the query interval is supposed to set as 1 hour for both BIND and Unbound resolvers. It is found that BIND update the keys strictly every one hour, however unbound are more sensitive to the change of keys in DNSKEY. It is inferred that UNBOUND update the key via alternative approach by listening to the normal DNSKEY query and response which are generated by monitoring purpose (query every 10 minutes). 


### Algorithm rollover without change on ZSK

We rolled the algorithm twice, we found the algorithm rollover with only KSK works for the resolver in Case 1 (with -P option) and Case 2. The BIND and Unbound can work with it without change on ZSK. 

### Configuration error in lower version of PowerDNS

PowerDNS (pdns-recursor 4.0.0~alpha2-2ubuntu0.1) resolver failed when configure multiple algorithm KSK.

### Stand-by Key

In the second algorithm rollover test, we deployed stand-by key by introducing the algorithm with two Key Signing Keys. The stand-by Key is not used to roll algorithm but used to sign in the absence of "incumbent" key in case of failure. The switch to stand-by key is smooth and swift without any error after the "incumbent" key is being silenced.

## Conclusion and suggestion for Future Yeti Algorithm Rollover

After two tests on algorithm rollover in lab environment, we have following conclusions:

* The dnssec-signzone of BIND is a handy tool to deploy DNSSEC for most cases. But for algorithm rollover, we should take care of the case when old ZSK is used to sign the DNSKEY record.

* Via Case 1 we learn the issue of Double-DS approach in algorithm rollover. And if anyone insist to roll the algorithm using Double-DNS approach, please use "

" option of dnssec-signzone. The risk is unknown so far.

* Case 2 prove the capability of rolling algorithm without ZSK. But the tests lack diversity of DNS resolver to provide a convincing result. It is worth to dig in depth. 

* Case 3 and Case 4 are expected to pass the tests with liberal and conservative approaches respectively. The tests just confirmed it.

* We also found that BIND and Unbound have some different behaviors.

As to the suggestions for future algorithm rollover in Yeti, we should invite more DNS resolvers. If you expected more findings please try case 2 or Case 1 with "-P" option using dnssec-signzone.
