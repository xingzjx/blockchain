# Swarm tech AMA - How to run Bee nodes

00:18
hello everybody if
00:20
everything is working out we should be
00:23
live on youtube
00:24
right now um we are bringing you oh let
00:27
me start by presenting myself my name is
00:29
michelle bleur
00:30
many of you might have encountered me in
00:33
the discord chat the last couple of days
00:35
so for all
00:36
those of you who i met um hello
00:39
and welcome we are here to do a
00:42
technical
00:42
ama this means that you can ask any
00:45
question
00:46
about the technicalities of b and the
00:48
swarm network because we have with you
00:51
um today we have the some people of the
00:54
developer team
00:55
we have enrique hendrickson for instance
00:57
the product owner of bee
00:59
so rincey feel free to well all of you
01:02
guys feel free to
01:03
unmute and show your faces
01:06
and cameras
01:11
hello there we go so if if everything is
01:14
working out like we have audio from
01:15
rinke
01:17
yeah that's correct so hi everybody my
01:20
name is uh link so
01:21
michelle she already said i'm the
01:23
product owner of the b
01:24
team and you find us in a very special
01:27
moment as you can imagine
01:28
with the launch of the wonders hero just
01:31
a few days
01:32
in front of us um but i'm very happy to
01:36
answer all your questions
01:37
so um that's me i don't know do you want
01:40
to introduce yourself danny
01:41
sure so my name is daniel and i'm one of
01:45
the architects of swarm
01:47
and i've been closely involved with
01:50
recently with the
01:51
design of the bonding curve and
01:54
the choice of the layer 2 solution for
01:57
the
01:58
for the microtransactions
02:01
and then we have gregor in the middle
02:05
uh hi everybody um i'm
02:08
the only let's say non-death here my
02:11
role
02:12
this call is to listen and see if there
02:15
are any questions that are not related
02:17
to the technical side
02:18
uh we'll take note of them and
02:24
[Music]
02:35
can you do that again
02:39
one second we need to victor should use
02:41
his microphone
02:48
um
02:50
this one together with some others uh
02:54
ask me anything okay we will definitely
02:58
do that and now we have a van dot from a
03:00
separate location
03:02
hi my name is evan van dutt i'm a system
03:05
engineer in charge of managing
03:08
this form infrastructure nice to
03:12
meet you thank you so much um
03:15
sorry yes can i just say if i'm not
03:17
we're so used to actually like like we
03:19
always address you with your last name i
03:20
can also say evan of course if you
03:22
prefer that
03:24
vandal is is okay
03:27
okay it just sounds so cool in in dutch
03:29
that's that's why i always do it
03:31
so guys let's right jump in because we
03:34
have limited time
03:35
and an amazing lot of questions um
03:38
i'm going to start with one for fun dot
03:41
as you are the
03:42
system and infrastructure engineer this
03:45
is a question we
03:46
you know it kept repeating and repeating
03:48
what are the hardware requirements
03:50
for a mainnet node so how can people
03:52
prepare and what kind of hardware should
03:54
they prepare
03:56
for running a node on mainnet
03:59
well yeah we didn't uh do uh uh
04:02
how to say proper testing you know to
04:05
to have exact requirements i know i know
04:09
requirements for the for the nodes that
04:11
we are running
04:14
boot nodes are a little bit more uh
04:18
system they need a little bit more
04:21
system performance but yeah regular
04:24
users will not
04:25
manage the boot nodes so
04:29
regular storage nodes are around
04:33
three gigabytes of ram and one or two
04:37
cpus depending on
04:38
how fast you want your node but
04:41
yeah those are the nodes that we have
04:44
managed
04:44
uh are around that
04:47
and yeah enough
04:51
space so a couple of gigs
04:54
and a couple of processors it's it's not
04:58
it's not very specific no i'm just i
05:01
know i know because people are like what
05:02
are the specifications and then we say
05:04
like very general things
05:05
but um what what should my hard disk
05:08
size be
05:09
and is it important even uh
05:13
it is important
05:14
[Music]
05:16
uh let me maybe answer this one if you
05:19
want fondant
05:20
sorry i can maybe answer about the heart
05:22
please
05:24
yeah so the hardest requirements like
05:27
they're about
05:28
well i'm good of my video i'm sorry yeah
05:30
i think we lost video yeah
05:31
yeah so the hottest um they're basically
05:34
three systems in b that
05:35
depends on the size of the hard disk so
05:38
the actual storage
05:39
of your b nodes is two systems so we
05:41
have the reserve
05:42
and we have the cache and we just did
05:45
before this call we did a couple of
05:46
calculations so the reserve has
05:50
a capacity of 2 to the power 23 chunks
05:54
which is about 34 gigabytes
05:58
um then there's the other like storage
06:01
size
06:01
um this is modifiable to the user so
06:05
but the default size of the cache is
06:06
about four gigabytes
06:09
and apart from that there's also the
06:11
storage requirements uh which is needed
06:13
to run
06:13
and operate the b node um and i would
06:16
say that this is about five as well
06:19
so that amounts to like 34 plus four
06:22
plus five plus a bit of like uh
06:26
you know just just a bit of extra i
06:28
would say 55 gigabytes
06:30
recommended uh storage styles
06:34
okay so we have three gigabytes of ram
06:37
two processors and around 60 gigs of
06:40
hard drive
06:41
maybe one question for one dot like what
06:43
kind of hard drives are you running
06:45
like is there a requirement how fast
06:47
they need to be
06:49
if if you yeah if you use ssds
06:52
it's better it will be more performant
06:54
but
06:55
yeah so in in the production cluster we
06:58
are using
07:00
ssds
07:02
okay um okay so i'll just
07:06
again i throw in also like personal
07:08
questions i it's like ama right so i can
07:10
ask questions too
07:11
so i've been running um notes on
07:14
raspberry pis
07:15
with like a fast ssd will this still
07:19
work on mainnet has something like
07:21
really changed
07:22
or can i just keep running these notes
07:24
as they are
07:27
i mean for everything as a node operator
07:31
you are incentivized to run your node in
07:33
a proper way
07:34
so i would i didn't try running
07:37
raspberry pi's myself um so i would
07:40
advise you personally
07:41
just run them and see how they behave if
07:44
they work fine
07:45
they work fine awesome i'll i'll
07:48
definitely report back
07:49
you know on my finance there another
07:52
thing i just wanted to note about this
07:54
so of course like we are the developer
07:55
team right and the developers
07:57
are developing the software so we are
07:59
not necessarily experts in
08:00
running and operating them um so for
08:04
that we have a node operators channel in
08:06
the discord so
08:07
all these kind of questions uh if you
08:09
are a node operator
08:11
um please join this node operators
08:13
question and share the knowledge like
08:15
share
08:15
kind of the knowledge that you're using
08:17
for for running or we knows like what
08:18
works what doesn't work like we have
08:20
some answers
08:21
but i'm pretty sure that there are
08:23
people in the community that know
08:25
these kind of questions even better than
08:26
we do we do so
08:29
yes and there is wisdom in numbers or
08:31
something like that so
08:33
you know the more people that try it the
08:34
more we will figure out
08:36
if it works good next question um
08:39
how is the transition to mainnet going
08:42
to work with the current test nodes
08:44
will there be an upgrade path or will we
08:46
have to start from scratch
08:48
and what about the data what about the
08:50
data and what about
08:52
docker compose deployments it does a lot
08:55
of questions we'll start with one is
08:56
there
08:57
like a migration tool or you know how
09:00
how do we go i've been running notes on
09:03
test net now we're going to mainnet
09:05
so no is there an upgrade path
09:10
there will be a new network the short
09:12
answer
09:13
the testnet continues operating uh we
09:15
are rolling out
09:16
hopefully today a new version uh the 1.0
09:20
release candidate to the testnet
09:22
there will be an upgrade path for those
09:24
nodes that have been operating 0
09:26
6 nodes to 1.0 nodes okay
09:29
but if you want to operate a node
09:32
connected to the mainnet with real bcz
09:34
tokens you will be connecting
09:36
to a new network okay but so so but the
09:38
idea let's
09:39
let's stick with the test net for a
09:41
second so what does this mean that there
09:42
is like a a migration path
09:45
from i've been running like a test net
09:47
node on on oh
09:48
that was like 1.0 point something now
09:51
i'm going to this new 1.0 release
09:53
candidate how does it work can i just
09:55
overwrite it
09:56
or yes basically yes
09:59
so so so the the easiest thing to
10:03
imagine is is
10:05
if you imagine that the data that that
10:07
swamp stores
10:08
is not only the data itself in a
10:10
particular format
10:12
which even if it happens to be the same
10:15
then then it cannot be upgraded because
10:18
because
10:19
uh because it's not only the data it's
10:21
it's the proofs of
10:22
of of uh the story it's positive stamps
10:25
like
10:26
proof of proof of property
10:29
yeah in attached to it and and that is
10:33
uh basically checked against
10:36
a particular ethereum
10:40
chain that that it's running on so so in
10:42
the main that it's very different from
10:44
from the test at once so so and then
10:47
this is this is not
10:49
interchangeable and they're not not
10:50
upgradable and
10:52
and so so so the best i can recommend if
10:55
you want to
10:56
if you have some data that you you
10:58
currently have on the test
10:59
this net node or test on test net then
11:02
then the best
11:03
you can do is download and upload on the
11:05
new one
11:07
okay i'll bring that attach attach the
11:10
postage stamps that
11:11
you need yeah i think that's quite clear
11:14
um um maybe i don't know
11:17
uh rinke who do you want to give this
11:19
question to but what about
11:21
docker compose deployments i i
11:24
i never did docker compose deployment so
11:26
i have no idea really how that works but
11:28
people are asking like what about docker
11:30
compose deployments in the context of
11:32
upgrading to this 1.0 node
11:34
i'm not sure that yeah it's
11:38
nothing different than everything that
11:40
ring explained for the regular nodes you
11:42
know
11:43
uh if you want to continue running on
11:45
testnet you will just update
11:47
the the docker image inside the topic
11:49
host but if you want to switch to
11:51
mainnet yeah you will have to
11:53
as lincoln said it's a completely new
11:55
network
11:56
so yeah you have to clean your storage
11:59
and
11:59
start from scratch on on the main net
12:03
okay awesome there is nothing in
12:06
particular to
12:07
docker compose to yeah the other
12:10
stuff sorry you understand yeah thank
12:13
you vandals um
12:14
i just wanted to mention that uh
12:16
specific questions
12:17
about updating test net zero six nodes
12:21
to test net 1.0 nodes
12:24
the communication that we're going to do
12:27
this will go out today
12:28
in a blog post an announcement and all
12:31
of this communication
12:32
also includes instructions on how to
12:34
update
12:36
it will be very similar process to
12:37
previous updates
12:39
yes that that should go out somewhere
12:41
like late afternoon
12:43
very late afternoon probably um
12:47
once again but once again this is this
12:50
is
12:54
itself can be can be upgraded so if you
12:58
run on the same or
12:59
you're not on the same network then then
13:01
the upgrade can work
13:02
but if if that's upgrading in the sense
13:05
that
13:05
the same node to run on a different
13:07
network is not gonna work so
13:09
if you want to run your node on the
13:10
mainnet then you need to
13:12
uh restart okay perfect let's not spend
13:15
more time on that because it's actually
13:17
quite
13:17
simple thing if i hear you explain it um
13:20
i would
13:21
love to now hear from danny danny you
13:24
can't
13:24
imagine how many questions we got about
13:27
the bundling curve
13:28
like people are i can't say this like
13:31
people are like
13:32
really really confused about it and and
13:35
gregor maybe you can also chime in
13:36
because it's not only like
13:38
how does the bonding curve work it's
13:40
also in relation to our token supply
13:42
and you know people especially after the
13:45
token sale people have been very
13:47
confused about you say you're gonna have
13:48
a market
13:49
what is it the total supply of this
13:51
amount of tokens but at the same time
13:53
you start saying you're gonna mint new
13:54
tokens with a bonding curve
13:56
and so this is like totally um
13:59
you know like uh weird for people to
14:01
understand so if you guys could
14:03
maybe shed some light on that i don't
14:05
know maybe we can first hear like
14:07
what is a bonding curve and why do we
14:09
use it and then maybe gregor
14:10
you can explain you know how it all
14:12
works with like the
14:13
what people like to call tokenomics
14:18
i would uh um
14:21
we uh i would suggest uh yes if donnie
14:24
can explain how the bonding curve works
14:27
but uh today on this call uh
14:30
unfortunately we cannot go
14:31
into concretely discussing the bus stop
14:34
itself
14:35
um and um we will address
14:39
these questions uh we will address these
14:41
questions we can address them separately
14:43
after the launch
14:44
but the info the information uh
14:48
also let's say connect the dots is out
14:50
there
14:51
so um we just would pass the microphone
14:54
to
14:55
uh to done it it's okay oh sure of
14:58
course of course and i'm
14:59
totally with you the information is out
15:02
there connecting the dots maybe people
15:04
should do that themselves but anyway
15:05
danny let's dive into
15:06
what is a bonding curve and why do we
15:09
have it
15:10
right so a bonding curve contract is
15:12
essentially a marketplace
15:15
where you can buy and sell bus tokens
15:19
and what it does it bridges the temporal
15:22
gap between buyers and sellers
15:24
so basically you can deposit
15:28
bus tokens and get out the collateral
15:32
or conversely you can deposit collateral
15:36
and get out bus tokens
15:38
even if there's no other person
15:42
on the other side of the trade yet so
15:44
basically
15:45
this is a tool to ensure that
15:49
bus tokens are always liquid so if you
15:52
need bus tokens
15:54
the bonding curve is the place where you
15:55
can always get them
15:58
and if you are running a commercial
16:01
operation
16:02
uh running a swarm node
16:06
and you need to uh you need to sell
16:09
part of your earnings then the bonding
16:12
curve is there for you
16:13
to do that anytime you don't need to
16:16
actually have
16:17
another person
16:20
or another entity that is willing to do
16:23
the opposite trade
16:25
but if there are and you are trading
16:27
through the bonding curve
16:29
the mechanics of it is not really
16:30
different from a regular market
16:32
so you won't notice any big difference
16:35
is
16:36
the user interface is just like any
16:39
uh instant swaps it has been pioneered
16:43
by a shapeshift i believe
16:44
this kind of ui and it's used by
16:49
all sorts of exchanges including
16:52
decentralized ones like uniswap
16:55
so it's the same user experience also
16:58
can
16:58
can we go sorry if i can interrupt um
17:01
can we
17:02
can we go into the extremes i think this
17:04
is this shed some clarity as if you do
17:06
that so
17:07
what happens if everybody sends their
17:09
bus tokens to this bonding curve
17:12
well in that case they will get all the
17:14
collateral out
17:16
and there's no more bus token so the bus
17:19
tokens
17:20
bus all the bus tokens get redeemed for
17:23
the collateral and that's it
17:25
and so but but if this would happen you
17:27
can just
17:28
go back put and die and you will get new
17:31
burst tokens that's correct
17:33
okay so so this is hard for people to
17:35
understand like
17:36
like you can go to a zero token supply
17:38
and then get back to like a lot of
17:40
tokens yes
17:41
okay what happens and now the opposite
17:44
scenario so what happens if
17:46
if if like like like you know like
17:48
hundreds of thousands of people
17:50
start by buying bus stoves on the bonnet
17:52
curve and then
17:53
you know is there an end to it or what
17:56
happens when when that scenario
17:58
there are two answers to this question
18:00
so the first answer
18:01
is that the price will increase quite
18:03
steeply
18:05
so people will probably run out of money
18:07
before any other
18:09
any other uh limitation kicks in
18:13
but actually there is a limitation built
18:16
into the smart contract to avoid
18:19
uh numerical overflows and other kind of
18:23
hacks
18:23
with numbers but i seriously doubt that
18:27
we will get anywhere close to it
18:29
so that just to to give you
18:32
an order of magnitude it's uh
18:35
roughly if the price of the bus token
18:38
will go
18:39
to roughly half a billion die then the
18:42
limitations will kick in
18:44
but i don't think that it will ever
18:45
happen but never say never danny but
18:48
anyway
18:49
so can you explain i hope this is okay
18:51
um gekko but can you also explain how
18:54
so i think people understand now uh how
18:58
like like a token supply can be like
19:00
sort of elastic
19:01
with the bundling curve but what is the
19:03
relation with price
19:05
how does the that's i guess that's like
19:08
the bonds right or or
19:10
like what how does this price how is the
19:12
price calculated
19:14
well uh so the bonding curve function is
19:19
is a monotonically increasing and quite
19:21
steeply increasing function
19:23
meaning that if you buy with from the
19:25
bonding curve and nobody is selling to
19:27
it
19:27
that the price is increasing very fast
19:30
and if you're selling that nobody is
19:32
buying
19:32
then it's decreasing also quite fast but
19:36
if there are
19:36
actually other people on the other side
19:39
of the trade you just
19:40
you don't happen to be there at the same
19:42
time
19:43
so if the trading is roughly balanced
19:45
then the price will be stable
19:48
and also the bonding curve kind of acts
19:51
as a
19:52
inertia meaning that
19:55
since it has a large amount of
19:57
collateral load in it
19:59
then in order to move the price you
20:02
actually need to do
20:03
a substantial amount of trades so what
20:06
usually
20:06
or what often happens to new tokens
20:10
is that on the two-sided markets they
20:12
are not
20:13
there's not a terrible lot of liquidity
20:17
and because of that the price is
20:19
fluctuating
20:20
very widely and
20:23
we hope that the bonding curve will kind
20:26
of attenuate
20:27
this situation and the
20:30
price will really reflect kind of the
20:34
long-term balance
20:35
in supply and demand
20:38
so just personal question this is has
20:41
anyone ever done this
20:42
before like like are there examples of
20:45
of this uh
20:46
yes so bonding curves have been
20:49
pioneered by the bancor corporation if i
20:52
recall correctly bancor project
20:56
uh they're actually using a very similar
21:00
bonding curve to
21:01
to the one that we have chosen so it's
21:04
we're not doing a lot of innovation
21:06
there we changed a few parameters
21:09
because of various practical reasons
21:13
which i'm not ready to discuss right now
21:16
but uh
21:20
but yeah so bonding curves are
21:21
relatively new technology but we're not
21:25
we're definitely not the first to try it
21:27
okay maybe
21:28
this will be my last question yes sorry
21:31
i mean
21:32
yes it's like the bonding curve we
21:34
believe adds stability
21:36
to the whole swarm ecosystem it also
21:38
provides
21:39
independent decentralized liquidity
21:42
which also again strengthens
21:44
uh swarm as a project and in a way
21:48
uh you know it's uh we see a lot of
21:51
projects in bedford
21:53
so like curve dot finance well it's a
21:56
curve
21:57
you know uh the where they where they
21:59
got the name from so there is a lot of
22:01
innovation in this space
22:03
um and uh i would say i would expect
22:06
that we will see more and more projects
22:08
uh in future also um doing
22:12
similar approaches yeah i totally agree
22:14
you can also already see it like in
22:16
research and in like
22:17
some even some github commits from other
22:19
projects that like most of them are
22:21
experimenting at least with with curves
22:24
and i
22:25
i am all for it one this is a very
22:27
technical question
22:28
danny from me to you why oh why
22:31
um do we have 16 decimals like every erc
22:35
20 compatible token has 18 decimals and
22:37
are you
22:38
are we trying
22:42
and it has to do with the bonding curve
22:45
so basically what we want to avoid
22:47
is that in a situation where most of the
22:49
bus tokens are sold back to the bonding
22:52
curve
22:53
we want to avoid the situation when some
22:55
tokens are actually free
22:57
when they cost zero and in order to
23:00
avoid that
23:02
in addition to the bank or function
23:04
there's a little linear
23:06
[Music]
23:07
so basically we need to add the same
23:10
amount of
23:11
atomic units to the price
23:15
in bus and in
23:18
in die and we want that
23:22
to be insignificant roughly one percent
23:25
and one percent is two orders of two
23:27
decimal orders of magnitude
23:30
so it's a purely technical decision
23:35
i hope it will not upset an upset
23:39
anything
23:39
no the power of 16 is still a very big
23:43
number
23:43
yes yes that's true um um
23:46
no it won't upset people i think it's
23:48
even a good thing because you can
23:50
recognize all the scam tokens because
23:51
they have 18 decimals
23:53
and so this is you know only now maybe
23:56
the scam tokens
23:57
learns that we have 16 decimals so like
24:00
if you see a pcz token with 16 decimals
24:02
that doesn't mean
24:04
that this one is failed right right you
24:05
will see one with 18
24:07
that's a scam for sure yeah exactly yeah
24:09
yeah
24:10
exactly thank you rinky for the
24:11
disclaimer um
24:14
so i'll go back to my list of questions
24:18
here thank you so much danny that's that
24:20
was like a very thorough explanation i
24:22
think
24:23
um this is a this is one that
24:26
probably for you has the over generous
24:29
free time
24:30
settlement issue being fixed i have no
24:32
idea what this means but
24:34
maybe some of you guys do
24:37
um yeah so i'll provide a bit of
24:40
background so this is about the
24:42
um the ratio at which
24:45
nodes give each other uh free bandwidth
24:48
that is the
24:49
the time refreshment ratio um it was set
24:52
at a certain
24:53
uh parameter and then we increased the
24:56
third we interrupt but
24:57
you have to explain this what does this
24:59
mean what what does this mean even like
25:01
notes give each other time
25:03
that sounds very romantic but can you
25:05
explain that
25:07
yeah so this is the feature of
25:09
time-based sentiments right
25:10
um as you know before swarm nodes
25:14
all gave each other bandwidth in
25:16
exchange for money
25:19
and we realized a couple of months ago
25:21
actually that
25:23
this kind of creates a critical
25:25
dependency
25:26
from the swarm network to the blockchain
25:28
so if for whatever reason
25:30
the blockchain wouldn't operate or
25:34
nodes maybe would not like trust the
25:36
blockchain anymore
25:37
for security reasons or whatever then
25:40
this whole swarm network would fall
25:41
apart and we don't want it
25:43
um and we figured out that
25:48
we do want to keep these monetary
25:50
settlements
25:51
but alongside that we introduced a
25:53
different kind of sediment layer which
25:55
we call time-based settlements
25:57
so the time-based settlements is
25:59
basically nodes giving each other
26:01
a certain kind of bandwidth capacity
26:05
for free so they don't need to pay for
26:07
that and and what you see then
26:08
is that the two most important reasons
26:10
for the swap bandwidth incentives
26:13
is basically spam protection and
26:16
um to incentivize the nodes collaborates
26:19
and
26:19
with time-based settlements they are
26:22
preserved
26:24
so the time based settlements is there
26:26
if notes don't
26:28
create like don't use a lot of bandwidth
26:31
but if you would
26:32
like start download the movie or
26:34
whatever then you
26:35
consume more than time based settlement
26:37
and you use the monetary settlements
26:40
um so in the p06
26:43
the time-based settlements basically
26:45
nodes gave each other too much
26:46
free bandwidth and this has been reduced
26:49
now with the 1.0 release candidate
26:51
and yeah we're going to run out today
26:53
and and see how it behaves
26:56
okay so the notes were too generous
26:59
exactly okay cool so uh this is an
27:02
interesting question i think
27:04
what about the high gas prices on
27:06
ethereum
27:07
is an off chain or level 2 solution
27:09
planned
27:12
so let me answer that so we have
27:15
researched
27:18
the alternatives some layer two
27:19
solutions and
27:21
basically we settled on
27:24
a side chain called hexadiene
27:27
x-die is a evm-based side chain which
27:31
of ethereum meaning that it has this
27:34
exact same smart contract mechanics as
27:36
ethereum itself
27:38
uh same transaction format
27:41
same kind of apis which means
27:45
that the transition is relatively
27:47
painless
27:49
it has been operating for a while so it
27:51
has a good track record
27:52
it is used by other projects so
27:56
so there's already some community trust
27:59
around it
28:00
uh it is cheaper and faster than
28:03
ethereum may not and it can carry the
28:06
kind of microtransactions
28:07
that swarm requires but it's not a long
28:10
term solution
28:11
so this is a stop gap measure because
28:14
basically ethereum scaling problems
28:17
didn't catch up with
28:19
one's requirements yet but we're really
28:22
hopeful that
28:24
there will be some standard scalability
28:26
solution for ethereum
28:28
maybe ethereum two maybe some roll up
28:31
we don't know yet but when it comes
28:34
around we can
28:35
we can easily change that so x-die
28:39
is a fully bridged uh side chain
28:42
which means that bus tokens can move
28:45
from the main to the side chain and back
28:48
trustlessly and and safely
28:52
and we're only committed to using the
28:55
bus token
28:56
but where these bus tokens live is
29:00
it it can change depending on how
29:01
technology develops
29:03
and maybe for example if swarm's
29:05
popularity explodes
29:07
maybe it will be the very reason why xdi
29:10
network will be clogged up
29:12
so it's not an ultimate scaling solution
29:15
but we think that for the
29:17
time being which means months and maybe
29:20
even years
29:22
from now it's an adequate solution so
29:25
maybe i want to add like one
29:26
clarification you say that xdi
29:28
people can move their tokens trustworthy
29:31
i don't think that's correct
29:33
because we we still depend on the bridge
29:35
operators to
29:36
uh to migrate the tokens so there is a
29:39
certain trust in the operators of the
29:41
bridge
29:42
yes but they are in something else they
29:44
are incentivized
29:46
yeah yeah but um it is
29:49
trustless to a certain degree but not as
29:51
trustless as the imperial mainland
29:53
rights
29:54
exactly but yes but at least we
29:57
as a team say we trusted so in that way
30:00
you can trust it
30:04
right yes okay um
30:07
okay wow what is this what is the
30:10
minimum
30:10
expected storage performance for b
30:14
is it a good idea to use nfs
30:17
or distributed block storage to store
30:20
chunks of multiple instances
30:23
are are nvme ssds the best overall
30:26
option does anyone know you know all
30:28
these terms
30:30
probably you guys should know this right
30:31
no definitely this
30:33
is the best option
30:36
but the first one i think you mentioned
30:38
nfs
30:40
yeah nfs are distributed block storage
30:42
like menio
30:46
yeah we didn't test it with
30:50
menio it works with
30:53
ebs that's a
30:56
on on aws but yeah minio and
31:00
nfs really natural depending on the
31:03
network
31:06
speed on the disk speed again yeah with
31:09
nfs servers so yeah
31:11
but yeah faster the better but
31:15
we didn't tested it as rinka said
31:18
yeah it would be nice also community
31:22
yeah just test it and yeah wrap your
31:24
results in the node operator
31:26
yeah yeah thank you so much for that
31:29
it's it's fun to be able to trust on
31:31
your knowledge because i had no idea
31:33
what that all means
31:34
um are there limits to the storage space
31:38
a single b
31:39
can take up the db capacity option was
31:41
removed in 0.6.0
31:44
is there another way
31:47
yeah thank you okay so okay so
31:51
so the the this capacity was removed
31:54
because
31:55
because it's no longer it's no longer
31:58
one
31:59
parameter that that
32:02
that is that is used but there's a
32:05
reserve and the cache
32:06
and the reserve is is a is a system-wide
32:09
constant it's it's supposed to be the
32:11
same
32:12
reserve storage size for each each and
32:15
every node so in that sense it's not
32:16
configurable that's
32:18
that's that's it's actually let's say
32:21
let's see that's a part of the
32:22
requirement of running a node that you
32:24
that you have that space
32:26
and as for the cache the cache itself is
32:29
is is is configurable and and it can be
32:32
bigger
32:33
however however it's it it
32:37
it's depending depending on the
32:39
depending on the popular
32:40
population of the of the of the network
32:43
and and the number of uh chunks in it
32:46
the number of the volume of data in it
32:49
is
32:50
it is it's going to be after after first
32:53
certain size
32:54
it's not going to be more profitable to
32:56
to operate a bigger bigger cache
32:58
uh most most likely so so there's
33:02
this practically there's there's no gain
33:04
in having
33:05
having a lot of storage for one node
33:08
and and i hope it answers the question
33:11
and
33:12
and just just landed the side notes here
33:14
so there's there's in this form there's
33:15
such a thing as pinning
33:17
that certain certain content that you
33:19
can you can
33:20
pin to your local node and and and have
33:23
it
33:24
as part of your part of your um
33:28
storage that is not removed by the by
33:30
the by the garbage collection processes
33:34
but this is important to know that this
33:36
pin content
33:37
is is uh calculated on top of
33:40
the other requirements so if you want to
33:42
pin content like lots of
33:44
movies you have to take care that the
33:46
space for those pinned content
33:48
is available unless unless the pin
33:50
content is otherwise protected
33:52
in which case it's part of the the
33:53
reserve anyway so so once again if you
33:56
have postage stamp
33:58
if you if you have if you have content
34:00
which you want to pin
34:01
with expired postage stamps you need
34:04
you need to take care that that the the
34:07
storage
34:08
space that it takes up is is extra and
34:10
provided over over on top of the the
34:12
questions
34:14
i hope it's clear that totally makes
34:16
sense to me
34:17
but i did read the book of swarm um
34:20
this is i want to add like a small
34:21
clarification here
34:23
if i may so the reserve
34:26
is for those chunks that you're supposed
34:28
to store
34:29
based on like your proximity to the
34:31
chunks so that's what we call the
34:32
natural location in the white paper
34:34
so all the chunks that you are like
34:36
storing based because they are so close
34:37
to you
34:38
they go to the reserve and apart from
34:41
that note operators also
34:43
what we call cash chunks whenever i am
34:46
forwarding a chunk to his destination
34:48
and this caching is actually
34:50
incentivized because if i if i'm
34:52
cashing a chunk and i get requested it
34:55
means that i don't need to pay
34:56
to get it to my drone stream here but i
34:58
can immediately set it and have pure
34:59
profit
35:00
and this is the cash right so that's why
35:03
the cash is modifiable
35:05
like it doesn't really matter if you
35:06
cash or not like the network will
35:08
still function if no note caches that's
35:10
why it doesn't matter
35:12
but the reserve capacity really should
35:14
stay the same
35:15
because this is yeah this is what you're
35:17
supposed to store based on your location
35:20
okay let's let's uh go on what is a
35:23
non-mineable
35:24
overlay and how exactly is it used
35:28
again i have no idea what what they mean
35:30
by that yeah
35:31
so i think i can answer that um whoever
35:33
asked this question is is keeping a very
35:35
close eye
35:36
to our pr so this vr is out there i
35:39
think maybe it's merged right now
35:40
maybe it will be merged in the upcoming
35:43
hour but
35:44
um basically swarm has a requirement
35:48
and we are always reasoning about swarm
35:50
that node operators kind of like
35:52
randomly choose
35:53
their overlay autos right and then we
35:56
know that all artists
35:57
are good sorry an overlay address is the
36:00
address like the ip address of the note
36:02
no it's it's the um it's the address
36:05
that basically defines what chunks
36:06
you'll be storing
36:08
okay it's similar to the ethereum
36:10
address
36:12
yeah so
36:15
we're always assuming that the overlay
36:18
artists are
36:19
evenly distributed around the network
36:22
however um the overlay artist before was
36:25
purely
36:26
based on the ethereum honors of the user
36:30
so by choosing like by kind of like what
36:32
you call mining
36:33
uh trying out many private keys i could
36:36
try out like so many times
36:38
until i have a particular overlay
36:40
address that would land into a certain
36:41
neighborhood
36:42
now this opens all kinds of bad
36:45
scenarios maybe the very first one
36:47
just that the overlay offices are not
36:48
evenly distributed so one space of the
36:50
network has more capacity than another
36:52
space as
36:53
network but one could also think about
36:55
more kind of adversarial scenarios where
36:58
i would just
36:58
kind of mine thousands of others in one
37:01
neighborhood and thereby kind of
37:03
completely
37:04
isolating that neighborhood from the
37:05
network
37:07
so this is where the non-mineable
37:09
overlay obviously
37:10
comes in so basically we added one
37:12
additional ingredients
37:13
um what defines what your overlay iris
37:16
is going to be
37:17
and this is the block hash at which your
37:20
notes
37:20
registered so basically now in order to
37:24
choose a neighborhoods i need to do a
37:27
transaction
37:28
on the die test chain so this will cost
37:30
real money
37:32
so node operators can still choose kind
37:35
of
37:35
what not the diet test chain right you
37:37
don't need to pay for that
37:39
yeah so you say like you have to do a
37:40
transaction on the diet test chain what
37:42
what that what do you mean by that
37:45
side chain chain okay just i'm just
37:47
correcting you because you know
37:49
you know these details matter um okay
37:52
this is uh to me it's at least clear i
37:54
hope also to the
37:55
author of that question um not operators
37:58
nothing changes like they don't
38:00
maybe the only thing they notice is that
38:01
the overlay artist is different
38:04
but apart from that this is really you
38:06
know something
38:07
internal under the hood um the next
38:10
question
38:11
can the swap initial deposit be adjusted
38:14
and if so what is the allowed range this
38:16
was
38:17
of course a question um you have to
38:19
explain this because
38:21
especially i think from china we get a
38:23
lot of questions about
38:25
um staking mining you know all that kind
38:28
of questions where we actually
38:29
see like this is just not true um
38:32
so if we tell people like you need to
38:35
put in
38:36
some bus right you know in order to
38:39
operate your node
38:40
is this true is this not you know can
38:42
you can you talk about that because this
38:44
is this is
38:44
this is something that was very
38:45
confusing no the answer is a resounding
38:49
oh some of you have to mute yeah
38:53
so the answer is guys please
38:57
oh god
39:00
okay try again so the answer is a
39:03
resounding no
39:05
you don't need to state buzz in order to
39:07
operate a node you can start with zero
39:09
and you can earn your way
39:12
into the network by providing the
39:14
service
39:16
i'm gonna double check this is this this
39:18
this is so i
39:20
i'm gonna restate this right i can start
39:23
a b note
39:24
i don't have to put in any bus i don't
39:26
have to buy bus
39:27
i can just run it and earn bus with it
39:33
um a small side note yes yes with zero
39:38
in theory you can earn your money to get
39:41
in and actually all the smart contracts
39:43
are completely prepared
39:44
uh that instead of note operators paying
39:47
you uh node operators can pay you by
39:50
deploying a checkbook for you
39:52
however this functionality needs a bit
39:54
more wiring in and we'll
39:55
we're planning um like a non-blackboard
39:57
compatible update
39:58
that does this at a later point but
40:02
the requirement to join the swarm with
40:05
zero bzz is perfectly possible and i
40:08
know michelle we have been having a
40:10
discussion about this like two months
40:11
ago
40:12
but this now is very easy especially
40:14
because of the time based settlements
40:15
that we spoke about before
40:17
so this is no no problem anymore and uh
40:20
yeah okay
40:20
so i'm gonna ask this once once again
40:23
just to have it
40:24
clear so i can start a b note and start
40:27
earning bcc without putting in any bzz
40:32
now today i will on the on june 21st
40:35
about the earning bcc i'm not too sure
40:38
because
40:38
it might be but i don't know if note
40:40
operators are
40:42
paying always to a checkbook or if they
40:44
can also pay
40:45
to an ethereum address so like
40:48
anyways if this is not enabled right now
40:51
it will be like
40:52
super easy to enable at a later point um
40:55
yeah maybe yeah
40:58
yeah gregor thank you um there was a
41:01
there was one i would also
41:02
ask here a question because a lot of
41:04
times i get another
41:06
question do does a node operator need to
41:08
deposit either
41:10
or x die in order to run the node
41:13
um the node operator um i mean like what
41:17
i said right you can be a node operator
41:19
without deploying a checkbook so in that
41:21
sense
41:22
no x die is needed as well however xl is
41:26
an
41:26
evm chain and as an evm chain you will
41:29
need
41:30
you will need guess the gas costs on
41:34
x die are significantly lower than the
41:37
guest cost on the ethereum main chain
41:39
and i even believe
41:40
that the xdi team is operating a faucet
41:44
for getting x die for your transaction
41:47
costs
41:50
so if i understand it correctly it's
41:52
more practical
41:54
and easier if there's like a little bit
41:56
of exercise
41:58
on the account yes yes yeah if you want
42:01
to deploy your checkbook if you want to
42:02
cash out your checks
42:04
like all these operations if you want to
42:05
do an operation on the blockchain
42:08
with your checkbooks with your caches
42:09
with your like
42:11
uh buying a postage stamp you will need
42:13
xdi
42:14
alongside vcc like a small amount to pay
42:16
for the guest piece
42:20
thank you thank you um this this is this
42:23
is i hope this sheds clarity because
42:25
there were so many
42:27
you know confused people asking about
42:29
speaking and you know
42:30
there are a lot of stories going around
42:33
and maybe i can promise an answer
42:35
maybe i can promise an answer for the
42:37
question whether not operators will be
42:39
able to earn bcc
42:40
without even having bcc i can promise
42:42
this answer like i need to check this
42:45
okay no problem i'll ask you again
42:46
monday at the live event
42:48
so yes please greg or go ahead
42:54
um maybe maybe it's a good thing
42:58
there are inferiors uh also mentioned
43:02
and uh for that there will be uh
43:05
needed speaking right and this maybe is
43:08
for those who read the book of swan can
43:10
also be
43:11
the source of confusion maybe maybe
43:13
victor you can
43:14
shed more light on this okay so
43:18
so indeed indeed indeed there's various
43:20
layers of
43:22
of of potential um
43:26
incentivization for for persistence and
43:28
persisting files
43:29
so the current version of swarm is only
43:32
going to implement
43:36
indirect positive rewards and
43:39
incentivization
43:40
this this is this means that that uh
43:43
those that that it's it's it's the
43:46
whole system has an emergent incentive
43:49
to to preserve
43:50
to preserve the files because he was
43:52
doing so you
43:54
you you comply if you comply with the
43:56
protocol then you're basically
43:58
serving serving the the your interest as
44:01
a bus token holder
44:02
and and uh and and instantly works
44:06
positively like that but the the
44:09
staking solution is is kind of is
44:11
usually under
44:12
uh so under under the under the heading
44:15
of
44:16
negative incentives which means that you
44:18
you
44:19
you have a mechanism that
44:20
disincentivizes you
44:23
to lose the contents prevents you from
44:25
from losing the content
44:26
by by the the threat of threat of some
44:30
some sort of confiscation or stress of
44:33
of of having you having your stay
44:36
forfeited
44:37
and and and that this this this layer of
44:40
insurance is not part of the current
44:43
release
44:44
but of course part of the part of the
44:46
release plan and then the roadmap
44:48
later and there's gonna be various
44:49
solutions for this
44:51
uh in the in the in the in the
44:55
in the in the in the middle of these two
44:57
solutions there's a
44:58
there's a there's a there's a
45:02
little solution which is which is the
45:04
direct positive rewards
45:05
which is a system which uh directly
45:08
redistributes uh some of the postage
45:11
stamp revenue so
45:12
the revenue that that is generated by by
45:15
people paying in
45:16
a kind of
45:20
rant-like
45:24
post for for that for the for storing
45:25
their files and in the same time
45:27
solution you
45:28
know in this institution
45:31
you you will get this this revenue
45:34
redistributed among
45:35
the the stores that that
45:38
that are entitled to it based on the
45:41
less
45:42
storage business storage activity so
45:46
this these are going to be the upgrades
45:48
on a road map
45:49
and and
45:52
so once again to wrap up there's no
45:54
staking currently needed
45:56
on the on the part of the boards uh
46:00
even the the the the amount that you
46:03
need to have is is is only to cover the
46:07
the cost of of having one transaction uh
46:10
attached to your
46:11
to your to the rest that that that is
46:15
the basis
46:15
for you for your swarm ideas for you
46:17
really this was the
46:19
minor overlay that that ring i mentioned
46:22
okay and and that's basically it
46:26
thank you for the for the for the
46:28
wrap-up
46:29
um this is um so guys we're gonna now
46:32
have like a fun round
46:36
um so imagine that you're in a tv show
46:38
now
46:39
um prometheus metrics that b exposes
46:42
are a bit obscure according to one
46:45
author
46:46
so this this uh this question oscar
46:49
is basically asking us to explain in
46:52
layman's terms
46:53
some terms that we use in the prometheus
46:56
um
46:57
interface are in what is it like the
46:58
logs or the debug interface
47:00
so um you can only answer if you can do
47:03
it in one sentence that's the rule of
47:05
the game okay so i'm gonna just say a
47:07
term
47:08
and then one of you has to answer it but
47:10
in one sentence
47:11
and with one sentence i mean like 15
47:13
seconds not like
47:14
not like a sentence that takes on and on
47:16
and on and on
47:17
okay get ready for this
47:22
accounting
47:24
i just want to say that the prometheus
47:26
metrics so far
47:28
has been like you know the users the
47:31
intended users of those are the
47:33
developers
47:33
um however of course they can also be
47:36
used by node operators um
47:38
but the prometheus metrics haven't been
47:41
kind of
47:41
sanitized to be useful for node for node
47:45
operators so maybe
47:46
does this mean you can't you don't want
47:48
to answer the question
47:49
or you don't
47:56
so i'll just i'll just but you have to
47:58
do it in one sentence
48:00
accounting accounting is there is the
48:03
peer-to-peer tracking of
48:07
bandwidth use between peers
48:10
yes that's a great sentence batch store
48:22
is the place where information about
48:25
uh about postage patty matches
48:29
are synchronized with the blockchain
48:32
okay
48:32
local store this is the store for chunks
48:37
and
48:37
one related information full storage
48:43
full storage yes
48:53
[Music]
49:01
the synchronization of chunks in within
49:04
the neighborhood
49:06
next one pull sync that's the
49:09
synchronization of shanks in the
49:10
neighborhood
49:12
for sure this is the component that
49:17
that that is running in the background
49:20
to make sure that chances that you
49:22
uploaded on your local road
49:24
reach the network and ultimately are
49:26
pushed to the
49:27
natural location
49:37
why didn't we call it just upload them
49:39
anyway that was not a question
49:42
retrieval download
49:48
swap
49:51
swap is the components that does the
49:55
peer-to-peer accounting to incentivize
49:57
for
49:57
for the bandwidth uh bandwidth account
50:00
okay
50:01
i that that was it thank you so much for
50:03
this first round
50:04
um i'm i'm so sorry that the floss
50:07
capacitor is not
50:08
in b anymore i was i was looking for
50:11
that because that was my favorite term
50:13
ever in in storm um but it's gone
50:17
so i don't know if i if i can do like a
50:20
request
50:20
but i think i'm gonna make an issue on
50:22
github like asking like can we just have
50:23
this back
50:24
because it's so funky anyway um
50:28
how much time do we have left uh casper
50:31
do i have to do we have to be like very
50:33
strict with time
50:34
or do we have more time
50:40
no we got maybe gasper already ran away
50:44
well anyway let's go to the next
50:46
question um
50:49
okay b seems to shred through
50:52
not boxes because of its high connection
50:55
count i
50:56
i had this is this likely to change so
50:59
so i was running so i'm running like b
51:02
notes on that note right
51:03
and when i just connect them in my home
51:05
network um
51:07
and i and i also then um watch a movie
51:10
for instance or i do
51:11
or anything my kids start i don't know
51:13
playing minecraft then the whole network
51:15
just crashes
51:16
like all the routers in the house just
51:17
say like this i'm gonna reboot
51:19
like so it's why
51:22
is this and will it be fixed
51:27
because of its high connection count is
51:29
there like is there like a limit on the
51:31
connection count
51:32
basically my isp is saying like what
51:34
you're doing is not
51:36
okay you're a hacker or something and
51:38
then they shut down my
51:42
incoming yeah i think i can answer it um
51:45
[Music]
51:47
b requires a certain number of
51:49
connection
51:50
per bin and a bin is like a term
51:52
specific for codemglia
51:54
i think right now it is eight
51:56
connections per bin
51:58
and every time the network doubles we
52:00
have another bin so i think
52:01
right now we have like 15 depths so that
52:04
means that there are two to the power of
52:06
15
52:07
uh nodes in the network is it correctly
52:08
victor
52:10
yes like and every time the network
52:14
doubles
52:14
your node would basically have eight
52:16
more connections to maintain
52:19
um now there are so certain like a
52:22
couple of edge conditions that nodes
52:24
would
52:24
connect to more than eight spears per
52:27
bin
52:28
and i would expect this to be sanitized
52:30
in the future that the node
52:32
is just maintaining those in in a better
52:34
way
52:36
um if you're really having issues with
52:39
that you can
52:40
manually manage and maintain all your
52:42
connections so there are debug api
52:44
endpoints where you can just say
52:46
disconnect to node so
52:48
to all the node operators that have
52:50
issues with that and are serious about
52:51
maintaining and operating your nodes if
52:53
they see
52:54
more than eight nodes in a particular
52:55
bin you can just
52:57
kick out any notes above eight
53:01
okay so the next question i i know it's
53:05
it's meant as a yes it's meant as a
53:06
technical question but i want to ask
53:08
this to gregor
53:09
um maybe just add the previous one uh
53:14
from what is discussed with some other
53:16
uh people
53:18
does maybe your question relates to the
53:20
initial boot up
53:21
of the b note when when it's more
53:24
actively syncing with the network
53:27
maybe that's the case yeah so so
53:29
basically
53:30
then there would just need to be some
53:32
throttling on the initial
53:35
foot
53:38
yeah because uh when it's like fully
53:40
synced it all goes down so
53:42
anyway gregor good that you're talking
53:44
because i wanted to ask this question to
53:46
you specifically
53:47
um because of your cypherpunk background
53:51
um so someone is asking can i run uh
53:56
no a mainnet b note on and then it's
53:58
just like insert like aws or like uh
54:01
clouds what are they called like all
54:03
these cloud providers right
54:05
and so i know technically we already
54:07
explained you can run a mainnet note on
54:10
aws
54:11
but i want you to explain why this is
54:13
not
54:14
i mean if you really believe in what
54:15
swarm is trying to achieve why this is
54:17
maybe not the best idea
54:19
i mean from an ideological standpoint
54:22
yes of course
54:22
it's you know uh i guess it starts just
54:25
with this
54:26
simple phrase we all know my uh my clip
54:29
to my keys
54:30
you know and it also applies the same
54:32
logic uh
54:33
the same cycle path logic if you want to
54:35
say uh if you want to say so it's like
54:37
basically
54:38
the user repairing the control having
54:40
the agency so
54:42
by uh hosting b nodes uh on
54:45
cloud providers we kind of put
54:49
this control into third party scans so
54:52
well it might just work perfectly
54:56
it does not really add to the
55:00
the whole idea of decentralization in
55:02
this case you've seen
55:03
especially already this year really
55:06
amazing
55:07
like really surprising moves uh
55:10
uh how amazon was uh overnight cutting
55:14
off
55:14
uh some um ad providers and developers
55:17
and
55:18
things like that so uh of course we
55:21
would as
55:21
a small foundation of support and
55:24
endorse that people run
55:26
be nodes on their own hardware this is
55:28
also
55:29
i think that node is actually written
55:32
extensive on this topic because that not
55:34
this
55:36
issue yes i i always keep sending people
55:40
to
55:40
you know because people are often
55:43
looking at aws for instance for like
55:45
convenience
55:46
and i and i try to then show them uh
55:48
that with that note
55:49
it's also very convenient and you stay
55:52
within
55:53
like the you know the goal and the
55:55
vision of what swarm is trying to
55:57
achieve
55:59
um if others want to elaborate
56:02
sure yeah please do i think this is an
56:04
important it's not it's not a very
56:06
technical topic but i think in a way
56:08
our ideological philosophy behind swarm
56:10
is also very technical
56:12
um not for developers but like for
56:14
people into philosophy and ideology it's
56:16
it's quite
56:17
you know it's quite a deep dive also i
56:20
would have done it practically
56:23
so if you take this idea of
56:24
decentralization to the extreme
56:26
then then amazon might not be really the
56:29
most part of the solution
56:32
the whole point of swarm is that it it
56:34
does add
56:35
an extra layer of security and and and
56:38
and decentralization even in the face of
56:41
having having your node running
56:44
because at least it obfuscates all the
56:46
content specific
56:47
and an application specific uh
56:52
logic so so that if if a small node has
56:56
a basic infrastructure ingredient is
56:58
allowed to
57:00
run in general on on amazon
57:03
then on top of that there's no there's
57:05
no proper
57:06
potential mechanism to censor or ban or
57:10
drop
57:21
yeah i think i think that's i think
57:23
that's uh very very good that you say
57:25
that
57:25
daniel to make that distinction uh i
57:28
specifically wanted to talk not about
57:30
what we have today
57:31
but with what the vision is and where we
57:33
want to go and why then maybe aws is not
57:36
the best idea
57:37
i hope i hope i'm not opening a can of
57:39
worms here yeah but in
57:41
this anything it's like complete sense
57:42
it's this food like if amazon can can
57:44
shut down
57:45
the particular service of of an
57:48
application or a development team
57:50
but if if they want to shut down the
57:53
particular application or development
57:55
of an app on swarm then what they can do
57:58
is
57:58
to shut down all the small nodes
58:04
yeah now quite clear i think i hope this
58:07
was clear for
58:08
for our viewers i would i always send
58:10
people to
58:11
that want to understand this more i
58:13
always tell them like at least
58:14
read the introduction in the book of
58:16
swarm because you know it explains
58:18
so well um what the history is of what
58:22
we are trying to achieve and and why we
58:24
are doing it
58:25
um and i think that's a very important
58:27
aspect of swarm
58:28
that people need to understand if they
58:30
invest time and energy and bus stops and
58:33
running notes
58:34
um i'll take this as the last question
58:37
if that's okay with you guys um it seems
58:41
that downloads don't work correctly even
58:44
though the upload was marked as
58:46
completed
58:47
what gives and isn't this a critical
58:49
issue for mainnet
58:54
um yeah i mean of course they're asking
58:57
like
58:57
does it work yeah i mean this is this is
59:00
of course a very critical issue and
59:02
um it ties into like
59:05
what we've been working on basically for
59:07
the last year
59:09
um so the swarm is um
59:13
a rather complex piece of software that
59:15
has been operating
59:16
on hardware that we don't know and
59:18
really like operating all
59:20
across the globe um
59:21
[Music]
59:23
having all the bits and pieces fall into
59:25
place
59:26
is is rather complex and we have been
59:28
observing indeed as well
59:30
if you upload something to the network
59:32
then somehow you cannot get it back
59:34
um i'm very i'm rather confident
59:39
about the changes that we are going to
59:40
release today
59:42
and really curious to see
59:45
how the network will operate and perform
59:47
my own
59:48
observations have been that if i upload
59:51
a small size
59:52
i can download it again but if i upload
59:56
a bigger
59:56
size you know file usually i couldn't
59:59
download
60:00
it and this is because of you know just
60:03
statistics right so if i upload
60:05
1000 junks if one of those chunks is
60:08
missing
60:08
the file will be corrupt so really if
60:11
you upload bigger
60:12
sizes files all the chunks really need
60:15
to become in their place
60:16
and for this reason we advise the limits
60:20
on the upload size so we advise to do
60:23
swarm
60:24
in beginning with smaller size files
60:27
just because the
60:29
the probability of like being able to
60:31
retrieve all those jumps into those
60:32
files
60:33
is is higher and we work towards
60:35
increasing these limits
60:37
over the time okay that's that's a very
60:40
clear
60:41
um and comprehensive answer it reminds
60:43
me of
60:44
when i started to to do the internet
60:47
thing like at first it was even like
60:49
slow to download an image and you had to
60:51
wait like before
60:52
you know before it was was downloaded um
60:56
to me a lot of what we are doing reminds
60:58
me of
60:59
early days of the internet etc lsc we
61:02
are losing victor victor thank you so
61:03
much for this ama
61:07
okay and uh guys i want every one of you
61:11
so found out
61:12
there were not many questions about
61:13
infrastructure i'd see this as a good
61:16
sign
61:16
um and uh but thank you so much that you
61:19
wanted to be here today to to answer
61:21
these questions and just you know just
61:23
to show your face i think it's important
61:25
for our users and for node operators for
61:27
investors that they have the opportunity
61:29
to also meet
61:30
the team and to see that you know that
61:32
you guys are like
61:34
you know what you're talking about
61:35
basically um so
61:37
i want to thank you all rinke um gregor
61:40
danny as always um i'm gonna see you
61:43
guys really really soon in budapest i'm
61:45
looking forward to that
61:46
with that i would like to round up the
61:48
ama please uh
61:49
everyone viewing if your quest question
61:52
was not answered just go to our discord
61:53
channel
61:54
go to node operators to be support we
61:56
have dedicated channels
61:58
um where people will uh in a timely
62:00
fashion answer
62:01
your questions so for now i say goodbye
62:05
from antwerp and i'll see you soon in
62:06
budapest
62:07
[Music]
62:18
bye
62:34
you