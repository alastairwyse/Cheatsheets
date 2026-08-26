Thoughts

What is the goal / purpose? -> Explain how the live split and merge process works in a general way... with view to using techniques for other cases of live redistribution and scaling of stateful services.

Should write a forward/brief explaining these goals very clearly. Maybe make a clear distinction with scaling of stateless services (which is straightforward)... maybe explain complexities of moving data whilst maintaining some level of consistency, and hence continuing to respond to queries accurately.

Need to explain some rough background about what AppAccess is. Some history of it stemming from simple permission management library used for business apps, to becoming a vehicle for demonstrating good design practice, personal experimentation with frameworks (e.g. grpc), and scaling/redistribution.

Explain basic 3 node setup... since that’s the basis for a shard group.

Mention event sourcing as it’s also a key enabler for redistribution. Maybe refer back to ms article on the same... mention how AppAccess dB model allows full load at any point in time without having to reconstruct from events.

Talk about consistent hashing... how it’s applied.

Overall sharded design... distopcoord which is stateless and infinitely scalable... one writer and many reader... affinity between writer and it’s database.  Event caching and full load process if event cache gets behind. Maybe a little bit about event buffering and dB writing in writer.

Could remove affinity between shard group and dB... dB could be independently shardable... but would need to have shard ranges included in load() queries to dB.
2025-12-20 - maybe for ‘alternatives’ section as proposed below

Go back through ‘feature’ list in AppAccess todo and see if anything else there is relevant to write about.

Explain generally about approach of moving previous events in batches and then small hold period when doing final switchover... similar to VMware (and try to find others as well).

Maybe discuss troubles with previous approach of trying to merge into existing... maybe go into pluses and minuses of having dedicated tables for primary elements.

Also think about some of the base features of AppAccess which make redistribution easier... like maybe the rest exception Jason format and rethrowing of exceptions... anything like that which aids redistribution.
Maybe have a precursors/requirements section... including this exception rehydration... hmmm is that a precursor for splitting tho??

Talk about data model... pure event sourced vs traditional- storing only current state... being able to regenerate events

Possibility of merging into existing node... may have been possible if using mongo style dB model with no lookup tables

Maybe good to have an ‘alternatives’ section to discuss things like merging into existing node as mentioned above

DistOpRouter and shard Config... how they’re used to dynamically adjust node Config and routing... ability to pause etc...

General approach to scaling in distributed design... writers by sharing, readers by replicas.

Maybe mention about how locality of readers could be adjusted... I.e. put them physically near the client application to minimize network latency.

Explain how events are cached and then read by reader within a shard group