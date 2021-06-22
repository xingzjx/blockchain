# Swarm tech AMA - How to run Bee nodes <br>

 <br>00:18 <br>
hello everybody if <br>
00:20 <br>
everything is working out we should be <br>
00:23 <br>
live on youtube <br>
00:24 <br>
right now um we are bringing you oh let <br>
00:27 <br>
me start by presenting myself my name is <br>
00:29 <br>
michelle bleur <br>
00:30 <br>
many of you might have encountered me in <br>
00:33 <br>
the discord chat the last couple of days <br>
00:35 <br>
so for all <br>
00:36 <br>
those of you who i met um hello <br>
00:39 <br>
and welcome we are here to do a <br>
00:42 <br>
technical <br>
00:42 <br>
ama this means that you can ask any <br>
00:45 <br>
question <br>
00:46 <br>
about the technicalities of b and the <br>
00:48 <br>
swarm network because we have with you <br>
00:51 <br>
um today we have the some people of the <br>
00:54 <br>
developer team <br>
00:55 <br>
we have enrique hendrickson for instance <br>
00:57 <br>
the product owner of bee <br>
00:59 <br>
so rincey feel free to well all of you <br>
01:02 <br>
guys feel free to <br>
01:03 <br>
unmute and show your faces <br>
01:06 <br>
and cameras <br>
01:11 <br>
hello there we go so if if everything is <br>
01:14 <br>
working out like we have audio from <br>
01:15 <br>
rinke <br>
01:17 <br>
yeah that's correct so hi everybody my <br>
01:20 <br>
name is uh link so <br>
01:21 <br>
michelle she already said i'm the <br>
01:23 <br>
product owner of the b <br>
01:24 <br>
team and you find us in a very special <br>
01:27 <br>
moment as you can imagine <br>
01:28 <br>
with the launch of the wonders hero just <br>
01:31 <br>
a few days <br>
01:32 <br>
in front of us um but i'm very happy to <br>
01:36 <br>
answer all your questions <br>
01:37 <br>
so um that's me i don't know do you want <br>
01:40 <br>
to introduce yourself danny <br>
01:41 <br>
sure so my name is daniel and i'm one of <br>
01:45 <br>
the architects of swarm <br>
01:47 <br>
and i've been closely involved with <br>
01:50 <br>
recently with the <br>
01:51 <br>
design of the bonding curve and <br>
01:54 <br>
the choice of the layer 2 solution for <br>
01:57 <br>
the <br>
01:58 <br>
for the microtransactions <br>
02:01 <br>
and then we have gregor in the middle <br>
02:05 <br>
uh hi everybody um i'm <br>
02:08 <br>
the only let's say non-death here my <br>
02:11 <br>
role <br>
02:12 <br>
this call is to listen and see if there <br>
02:15 <br>
are any questions that are not related <br>
02:17 <br>
to the technical side <br>
02:18 <br>
uh we'll take note of them and <br>
02:24 <br>
[Music] <br>
02:35 <br>
can you do that again <br>
02:39 <br>
one second we need to victor should use <br>
02:41 <br>
his microphone <br>
02:48 <br>
um <br>
02:50 <br>
this one together with some others uh <br>
02:54 <br>
ask me anything okay we will definitely <br>
02:58 <br>
do that and now we have a van dot from a <br>
03:00 <br>
separate location <br>
03:02 <br>
hi my name is evan van dutt i'm a system <br>
03:05 <br>
engineer in charge of managing <br>
03:08 <br>
this form infrastructure nice to <br>
03:12 <br>
meet you thank you so much um <br>
03:15 <br>
sorry yes can i just say if i'm not <br>
03:17 <br>
we're so used to actually like like we <br>
03:19 <br>
always address you with your last name i <br>
03:20 <br>
can also say evan of course if you <br>
03:22 <br>
prefer that <br>
03:24 <br>
vandal is is okay <br>
03:27 <br>
okay it just sounds so cool in in dutch <br>
03:29 <br>
that's that's why i always do it <br>
03:31 <br>
so guys let's right jump in because we <br>
03:34 <br>
have limited time <br>
03:35 <br>
and an amazing lot of questions um <br>
03:38 <br>
i'm going to start with one for fun dot <br>
03:41 <br>
as you are the <br>
03:42 <br>
system and infrastructure engineer this <br>
03:45 <br>
is a question we <br>
03:46 <br>
you know it kept repeating and repeating <br>
03:48 <br>
what are the hardware requirements <br>
03:50 <br>
for a mainnet node so how can people <br>
03:52 <br>
prepare and what kind of hardware should <br>
03:54 <br>
they prepare <br>
03:56 <br>
for running a node on mainnet <br>
03:59 <br>
well yeah we didn't uh do uh uh <br>
04:02 <br>
how to say proper testing you know to <br>
04:05 <br>
to have exact requirements i know i know <br>
04:09 <br>
requirements for the for the nodes that <br>
04:11 <br>
we are running <br>
04:14 <br>
boot nodes are a little bit more uh <br>
04:18 <br>
system they need a little bit more <br>
04:21 <br>
system performance but yeah regular <br>
04:24 <br>
users will not <br>
04:25 <br>
manage the boot nodes so <br>
04:29 <br>
regular storage nodes are around <br>
04:33 <br>
three gigabytes of ram and one or two <br>
04:37 <br>
cpus depending on <br>
04:38 <br>
how fast you want your node but <br>
04:41 <br>
yeah those are the nodes that we have <br>
04:44 <br>
managed <br>
04:44 <br>
uh are around that <br>
04:47 <br>
and yeah enough <br>
04:51 <br>
space so a couple of gigs <br>
04:54 <br>
and a couple of processors it's it's not <br>
04:58 <br>
it's not very specific no i'm just i <br>
05:01 <br>
know i know because people are like what <br>
05:02 <br>
are the specifications and then we say <br>
05:04 <br>
like very general things <br>
05:05 <br>
but um what what should my hard disk <br>
05:08 <br>
size be <br>
05:09 <br>
and is it important even uh <br>
05:13 <br>
it is important <br>
05:14 <br>
[Music] <br>
05:16 <br>
uh let me maybe answer this one if you <br>
05:19 <br>
want fondant <br>
05:20 <br>
sorry i can maybe answer about the heart <br>
05:22 <br>
please <br>
05:24 <br>
yeah so the hardest requirements like <br>
05:27 <br>
they're about <br>
05:28 <br>
well i'm good of my video i'm sorry yeah <br>
05:30 <br>
i think we lost video yeah <br>
05:31 <br>
yeah so the hottest um they're basically <br>
05:34 <br>
three systems in b that <br>
05:35 <br>
depends on the size of the hard disk so <br>
05:38 <br>
the actual storage <br>
05:39 <br>
of your b nodes is two systems so we <br>
05:41 <br>
have the reserve <br>
05:42 <br>
and we have the cache and we just did <br>
05:45 <br>
before this call we did a couple of <br>
05:46 <br>
calculations so the reserve has <br>
05:50 <br>
a capacity of 2 to the power 23 chunks <br>
05:54 <br>
which is about 34 gigabytes <br>
05:58 <br>
um then there's the other like storage <br>
06:01 <br>
size <br>
06:01 <br>
um this is modifiable to the user so <br>
06:05 <br>
but the default size of the cache is <br>
06:06 <br>
about four gigabytes <br>
06:09 <br>
and apart from that there's also the <br>
06:11 <br>
storage requirements uh which is needed <br>
06:13 <br>
to run <br>
06:13 <br>
and operate the b node um and i would <br>
06:16 <br>
say that this is about five as well <br>
06:19 <br>
so that amounts to like 34 plus four <br>
06:22 <br>
plus five plus a bit of like uh <br>
06:26 <br>
you know just just a bit of extra i <br>
06:28 <br>
would say 55 gigabytes <br>
06:30 <br>
recommended uh storage styles <br>
06:34 <br>
okay so we have three gigabytes of ram <br>
06:37 <br>
two processors and around 60 gigs of <br>
06:40 <br>
hard drive <br>
06:41 <br>
maybe one question for one dot like what <br>
06:43 <br>
kind of hard drives are you running <br>
06:45 <br>
like is there a requirement how fast <br>
06:47 <br>
they need to be <br>
06:49 <br>
if if you yeah if you use ssds <br>
06:52 <br>
it's better it will be more performant <br>
06:54 <br>
but <br>
06:55 <br>
yeah so in in the production cluster we <br>
06:58 <br>
are using <br>
07:00 <br>
ssds <br>
07:02 <br>
okay um okay so i'll just <br>
07:06 <br>
again i throw in also like personal <br>
07:08 <br>
questions i it's like ama right so i can <br>
07:10 <br>
ask questions too <br>
07:11 <br>
so i've been running um notes on <br>
07:14 <br>
raspberry pis <br>
07:15 <br>
with like a fast ssd will this still <br>
07:19 <br>
work on mainnet has something like <br>
07:21 <br>
really changed <br>
07:22 <br>
or can i just keep running these notes <br>
07:24 <br>
as they are <br>
07:27 <br>
i mean for everything as a node operator <br>
07:31 <br>
you are incentivized to run your node in <br>
07:33 <br>
a proper way <br>
07:34 <br>
so i would i didn't try running <br>
07:37 <br>
raspberry pi's myself um so i would <br>
07:40 <br>
advise you personally <br>
07:41 <br>
just run them and see how they behave if <br>
07:44 <br>
they work fine <br>
07:45 <br>
they work fine awesome i'll i'll <br>
07:48 <br>
definitely report back <br>
07:49 <br>
you know on my finance there another <br>
07:52 <br>
thing i just wanted to note about this <br>
07:54 <br>
so of course like we are the developer <br>
07:55 <br>
team right and the developers <br>
07:57 <br>
are developing the software so we are <br>
07:59 <br>
not necessarily experts in <br>
08:00 <br>
running and operating them um so for <br>
08:04 <br>
that we have a node operators channel in <br>
08:06 <br>
the discord so <br>
08:07 <br>
all these kind of questions uh if you <br>
08:09 <br>
are a node operator <br>
08:11 <br>
um please join this node operators <br>
08:13 <br>
question and share the knowledge like <br>
08:15 <br>
share <br>
08:15 <br>
kind of the knowledge that you're using <br>
08:17 <br>
for for running or we knows like what <br>
08:18 <br>
works what doesn't work like we have <br>
08:20 <br>
some answers <br>
08:21 <br>
but i'm pretty sure that there are <br>
08:23 <br>
people in the community that know <br>
08:25 <br>
these kind of questions even better than <br>
08:26 <br>
we do we do so <br>
08:29 <br>
yes and there is wisdom in numbers or <br>
08:31 <br>
something like that so <br>
08:33 <br>
you know the more people that try it the <br>
08:34 <br>
more we will figure out <br>
08:36 <br>
if it works good next question um <br>
08:39 <br>
how is the transition to mainnet going <br>
08:42 <br>
to work with the current test nodes <br>
08:44 <br>
will there be an upgrade path or will we <br>
08:46 <br>
have to start from scratch <br>
08:48 <br>
and what about the data what about the <br>
08:50 <br>
data and what about <br>
08:52 <br>
docker compose deployments it does a lot <br>
08:55 <br>
of questions we'll start with one is <br>
08:56 <br>
there <br>
08:57 <br>
like a migration tool or you know how <br>
09:00 <br>
how do we go i've been running notes on <br>
09:03 <br>
test net now we're going to mainnet <br>
09:05 <br>
so no is there an upgrade path <br>
09:10 <br>
there will be a new network the short <br>
09:12 <br>
answer <br>
09:13 <br>
the testnet continues operating uh we <br>
09:15 <br>
are rolling out <br>
09:16 <br>
hopefully today a new version uh the 1.0 <br>
09:20 <br>
release candidate to the testnet <br>
09:22 <br>
there will be an upgrade path for those <br>
09:24 <br>
nodes that have been operating 0 <br>
09:26 <br>
6 nodes to 1.0 nodes okay <br>
09:29 <br>
but if you want to operate a node <br>
09:32 <br>
connected to the mainnet with real bcz <br>
09:34 <br>
tokens you will be connecting <br>
09:36 <br>
to a new network okay but so so but the <br>
09:38 <br>
idea let's <br>
09:39 <br>
let's stick with the test net for a <br>
09:41 <br>
second so what does this mean that there <br>
09:42 <br>
is like a a migration path <br>
09:45 <br>
from i've been running like a test net <br>
09:47 <br>
node on on oh <br>
09:48 <br>
that was like 1.0 point something now <br>
09:51 <br>
i'm going to this new 1.0 release <br>
09:53 <br>
candidate how does it work can i just <br>
09:55 <br>
overwrite it <br>
09:56 <br>
or yes basically yes <br>
09:59 <br>
so so so the the easiest thing to <br>
10:03 <br>
imagine is is <br>
10:05 <br>
if you imagine that the data that that <br>
10:07 <br>
swamp stores <br>
10:08 <br>
is not only the data itself in a <br>
10:10 <br>
particular format <br>
10:12 <br>
which even if it happens to be the same <br>
10:15 <br>
then then it cannot be upgraded because <br>
10:18 <br>
because <br>
10:19 <br>
uh because it's not only the data it's <br>
10:21 <br>
it's the proofs of <br>
10:22 <br>
of of uh the story it's positive stamps <br>
10:25 <br>
like <br>
10:26 <br>
proof of proof of property <br>
10:29 <br>
yeah in attached to it and and that is <br>
10:33 <br>
uh basically checked against <br>
10:36 <br>
a particular ethereum <br>
10:40 <br>
chain that that it's running on so so in <br>
10:42 <br>
the main that it's very different from <br>
10:44 <br>
from the test at once so so and then <br>
10:47 <br>
this is this is not <br>
10:49 <br>
interchangeable and they're not not <br>
10:50 <br>
upgradable and <br>
10:52 <br>
and so so so the best i can recommend if <br>
10:55 <br>
you want to <br>
10:56 <br>
if you have some data that you you <br>
10:58 <br>
currently have on the test <br>
10:59 <br>
this net node or test on test net then <br>
11:02 <br>
then the best <br>
11:03 <br>
you can do is download and upload on the <br>
11:05 <br>
new one <br>
11:07 <br>
okay i'll bring that attach attach the <br>
11:10 <br>
postage stamps that <br>
11:11 <br>
you need yeah i think that's quite clear <br>
11:14 <br>
um um maybe i don't know <br>
11:17 <br>
uh rinke who do you want to give this <br>
11:19 <br>
question to but what about <br>
11:21 <br>
docker compose deployments i i <br>
11:24 <br>
i never did docker compose deployment so <br>
11:26 <br>
i have no idea really how that works but <br>
11:28 <br>
people are asking like what about docker <br>
11:30 <br>
compose deployments in the context of <br>
11:32 <br>
upgrading to this 1.0 node <br>
11:34 <br>
i'm not sure that yeah it's <br>
11:38 <br>
nothing different than everything that <br>
11:40 <br>
ring explained for the regular nodes you <br>
11:42 <br>
know <br>
11:43 <br>
uh if you want to continue running on <br>
11:45 <br>
testnet you will just update <br>
11:47 <br>
the the docker image inside the topic <br>
11:49 <br>
host but if you want to switch to <br>
11:51 <br>
mainnet yeah you will have to <br>
11:53 <br>
as lincoln said it's a completely new <br>
11:55 <br>
network <br>
11:56 <br>
so yeah you have to clean your storage <br>
11:59 <br>
and <br>
11:59 <br>
start from scratch on on the main net <br>
12:03 <br>
okay awesome there is nothing in <br>
12:06 <br>
particular to <br>
12:07 <br>
docker compose to yeah the other <br>
12:10 <br>
stuff sorry you understand yeah thank <br>
12:13 <br>
you vandals um <br>
12:14 <br>
i just wanted to mention that uh <br>
12:16 <br>
specific questions <br>
12:17 <br>
about updating test net zero six nodes <br>
12:21 <br>
to test net 1.0 nodes <br>
12:24 <br>
the communication that we're going to do <br>
12:27 <br>
this will go out today <br>
12:28 <br>
in a blog post an announcement and all <br>
12:31 <br>
of this communication <br>
12:32 <br>
also includes instructions on how to <br>
12:34 <br>
update <br>
12:36 <br>
it will be very similar process to <br>
12:37 <br>
previous updates <br>
12:39 <br>
yes that that should go out somewhere <br>
12:41 <br>
like late afternoon <br>
12:43 <br>
very late afternoon probably um <br>
12:47 <br>
once again but once again this is this <br>
12:50 <br>
is <br>
12:54 <br>
itself can be can be upgraded so if you <br>
12:58 <br>
run on the same or <br>
12:59 <br>
you're not on the same network then then <br>
13:01 <br>
the upgrade can work <br>
13:02 <br>
but if if that's upgrading in the sense <br>
13:05 <br>
that <br>
13:05 <br>
the same node to run on a different <br>
13:07 <br>
network is not gonna work so <br>
13:09 <br>
if you want to run your node on the <br>
13:10 <br>
mainnet then you need to <br>
13:12 <br>
uh restart okay perfect let's not spend <br>
13:15 <br>
more time on that because it's actually <br>
13:17 <br>
quite <br>
13:17 <br>
simple thing if i hear you explain it um <br>
13:20 <br>
i would <br>
13:21 <br>
love to now hear from danny danny you <br>
13:24 <br>
can't <br>
13:24 <br>
imagine how many questions we got about <br>
13:27 <br>
the bundling curve <br>
13:28 <br>
like people are i can't say this like <br>
13:31 <br>
people are like <br>
13:32 <br>
really really confused about it and and <br>
13:35 <br>
gregor maybe you can also chime in <br>
13:36 <br>
because it's not only like <br>
13:38 <br>
how does the bonding curve work it's <br>
13:40 <br>
also in relation to our token supply <br>
13:42 <br>
and you know people especially after the <br>
13:45 <br>
token sale people have been very <br>
13:47 <br>
confused about you say you're gonna have <br>
13:48 <br>
a market <br>
13:49 <br>
what is it the total supply of this <br>
13:51 <br>
amount of tokens but at the same time <br>
13:53 <br>
you start saying you're gonna mint new <br>
13:54 <br>
tokens with a bonding curve <br>
13:56 <br>
and so this is like totally um <br>
13:59 <br>
you know like uh weird for people to <br>
14:01 <br>
understand so if you guys could <br>
14:03 <br>
maybe shed some light on that i don't <br>
14:05 <br>
know maybe we can first hear like <br>
14:07 <br>
what is a bonding curve and why do we <br>
14:09 <br>
use it and then maybe gregor <br>
14:10 <br>
you can explain you know how it all <br>
14:12 <br>
works with like the <br>
14:13 <br>
what people like to call tokenomics <br>
14:18 <br>
i would uh um <br>
14:21 <br>
we uh i would suggest uh yes if donnie <br>
14:24 <br>
can explain how the bonding curve works <br>
14:27 <br>
but uh today on this call uh <br>
14:30 <br>
unfortunately we cannot go <br>
14:31 <br>
into concretely discussing the bus stop <br>
14:34 <br>
itself <br>
14:35 <br>
um and um we will address <br>
14:39 <br>
these questions uh we will address these <br>
14:41 <br>
questions we can address them separately <br>
14:43 <br>
after the launch <br>
14:44 <br>
but the info the information uh <br>
14:48 <br>
also let's say connect the dots is out <br>
14:50 <br>
there <br>
14:51 <br>
so um we just would pass the microphone <br>
14:54 <br>
to <br>
14:55 <br>
uh to done it it's okay oh sure of <br>
14:58 <br>
course of course and i'm <br>
14:59 <br>
totally with you the information is out <br>
15:02 <br>
there connecting the dots maybe people <br>
15:04 <br>
should do that themselves but anyway <br>
15:05 <br>
danny let's dive into <br>
15:06 <br>
what is a bonding curve and why do we <br>
15:09 <br>
have it <br>
15:10 <br>
right so a bonding curve contract is <br>
15:12 <br>
essentially a marketplace <br>
15:15 <br>
where you can buy and sell bus tokens <br>
15:19 <br>
and what it does it bridges the temporal <br>
15:22 <br>
gap between buyers and sellers <br>
15:24 <br>
so basically you can deposit <br>
15:28 <br>
bus tokens and get out the collateral <br>
15:32 <br>
or conversely you can deposit collateral <br>
15:36 <br>
and get out bus tokens <br>
15:38 <br>
even if there's no other person <br>
15:42 <br>
on the other side of the trade yet so <br>
15:44 <br>
basically <br>
15:45 <br>
this is a tool to ensure that <br>
15:49 <br>
bus tokens are always liquid so if you <br>
15:52 <br>
need bus tokens <br>
15:54 <br>
the bonding curve is the place where you <br>
15:55 <br>
can always get them <br>
15:58 <br>
and if you are running a commercial <br>
16:01 <br>
operation <br>
16:02 <br>
uh running a swarm node <br>
16:06 <br>
and you need to uh you need to sell <br>
16:09 <br>
part of your earnings then the bonding <br>
16:12 <br>
curve is there for you <br>
16:13 <br>
to do that anytime you don't need to <br>
16:16 <br>
actually have <br>
16:17 <br>
another person <br>
16:20 <br>
or another entity that is willing to do <br>
16:23 <br>
the opposite trade <br>
16:25 <br>
but if there are and you are trading <br>
16:27 <br>
through the bonding curve <br>
16:29 <br>
the mechanics of it is not really <br>
16:30 <br>
different from a regular market <br>
16:32 <br>
so you won't notice any big difference <br>
16:35 <br>
is <br>
16:36 <br>
the user interface is just like any <br>
16:39 <br>
uh instant swaps it has been pioneered <br>
16:43 <br>
by a shapeshift i believe <br>
16:44 <br>
this kind of ui and it's used by <br>
16:49 <br>
all sorts of exchanges including <br>
16:52 <br>
decentralized ones like uniswap <br>
16:55 <br>
so it's the same user experience also <br>
16:58 <br>
can <br>
16:58 <br>
can we go sorry if i can interrupt um <br>
17:01 <br>
can we <br>
17:02 <br>
can we go into the extremes i think this <br>
17:04 <br>
is this shed some clarity as if you do <br>
17:06 <br>
that so <br>
17:07 <br>
what happens if everybody sends their <br>
17:09 <br>
bus tokens to this bonding curve <br>
17:12 <br>
well in that case they will get all the <br>
17:14 <br>
collateral out <br>
17:16 <br>
and there's no more bus token so the bus <br>
17:19 <br>
tokens <br>
17:20 <br>
bus all the bus tokens get redeemed for <br>
17:23 <br>
the collateral and that's it <br>
17:25 <br>
and so but but if this would happen you <br>
17:27 <br>
can just <br>
17:28 <br>
go back put and die and you will get new <br>
17:31 <br>
burst tokens that's correct <br>
17:33 <br>
okay so so this is hard for people to <br>
17:35 <br>
understand like <br>
17:36 <br>
like you can go to a zero token supply <br>
17:38 <br>
and then get back to like a lot of <br>
17:40 <br>
tokens yes <br>
17:41 <br>
okay what happens and now the opposite <br>
17:44 <br>
scenario so what happens if <br>
17:46 <br>
if if like like like you know like <br>
17:48 <br>
hundreds of thousands of people <br>
17:50 <br>
start by buying bus stoves on the bonnet <br>
17:52 <br>
curve and then <br>
17:53 <br>
you know is there an end to it or what <br>
17:56 <br>
happens when when that scenario <br>
17:58 <br>
there are two answers to this question <br>
18:00 <br>
so the first answer <br>
18:01 <br>
is that the price will increase quite <br>
18:03 <br>
steeply <br>
18:05 <br>
so people will probably run out of money <br>
18:07 <br>
before any other <br>
18:09 <br>
any other uh limitation kicks in <br>
18:13 <br>
but actually there is a limitation built <br>
18:16 <br>
into the smart contract to avoid <br>
18:19 <br>
uh numerical overflows and other kind of <br>
18:23 <br>
hacks <br>
18:23 <br>
with numbers but i seriously doubt that <br>
18:27 <br>
we will get anywhere close to it <br>
18:29 <br>
so that just to to give you <br>
18:32 <br>
an order of magnitude it's uh <br>
18:35 <br>
roughly if the price of the bus token <br>
18:38 <br>
will go <br>
18:39 <br>
to roughly half a billion die then the <br>
18:42 <br>
limitations will kick in <br>
18:44 <br>
but i don't think that it will ever <br>
18:45 <br>
happen but never say never danny but <br>
18:48 <br>
anyway <br>
18:49 <br>
so can you explain i hope this is okay <br>
18:51 <br>
um gekko but can you also explain how <br>
18:54 <br>
so i think people understand now uh how <br>
18:58 <br>
like like a token supply can be like <br>
19:00 <br>
sort of elastic <br>
19:01 <br>
with the bundling curve but what is the <br>
19:03 <br>
relation with price <br>
19:05 <br>
how does the that's i guess that's like <br>
19:08 <br>
the bonds right or or <br>
19:10 <br>
like what how does this price how is the <br>
19:12 <br>
price calculated <br>
19:14 <br>
well uh so the bonding curve function is <br>
19:19 <br>
is a monotonically increasing and quite <br>
19:21 <br>
steeply increasing function <br>
19:23 <br>
meaning that if you buy with from the <br>
19:25 <br>
bonding curve and nobody is selling to <br>
19:27 <br>
it <br>
19:27 <br>
that the price is increasing very fast <br>
19:30 <br>
and if you're selling that nobody is <br>
19:32 <br>
buying <br>
19:32 <br>
then it's decreasing also quite fast but <br>
19:36 <br>
if there are <br>
19:36 <br>
actually other people on the other side <br>
19:39 <br>
of the trade you just <br>
19:40 <br>
you don't happen to be there at the same <br>
19:42 <br>
time <br>
19:43 <br>
so if the trading is roughly balanced <br>
19:45 <br>
then the price will be stable <br>
19:48 <br>
and also the bonding curve kind of acts <br>
19:51 <br>
as a <br>
19:52 <br>
inertia meaning that <br>
19:55 <br>
since it has a large amount of <br>
19:57 <br>
collateral load in it <br>
19:59 <br>
then in order to move the price you <br>
20:02 <br>
actually need to do <br>
20:03 <br>
a substantial amount of trades so what <br>
20:06 <br>
usually <br>
20:06 <br>
or what often happens to new tokens <br>
20:10 <br>
is that on the two-sided markets they <br>
20:12 <br>
are not <br>
20:13 <br>
there's not a terrible lot of liquidity <br>
20:17 <br>
and because of that the price is <br>
20:19 <br>
fluctuating <br>
20:20 <br>
very widely and <br>
20:23 <br>
we hope that the bonding curve will kind <br>
20:26 <br>
of attenuate <br>
20:27 <br>
this situation and the <br>
20:30 <br>
price will really reflect kind of the <br>
20:34 <br>
long-term balance <br>
20:35 <br>
in supply and demand <br>
20:38 <br>
so just personal question this is has <br>
20:41 <br>
anyone ever done this <br>
20:42 <br>
before like like are there examples of <br>
20:45 <br>
of this uh <br>
20:46 <br>
yes so bonding curves have been <br>
20:49 <br>
pioneered by the bancor corporation if i <br>
20:52 <br>
recall correctly bancor project <br>
20:56 <br>
uh they're actually using a very similar <br>
21:00 <br>
bonding curve to <br>
21:01 <br>
to the one that we have chosen so it's <br>
21:04 <br>
we're not doing a lot of innovation <br>
21:06 <br>
there we changed a few parameters <br>
21:09 <br>
because of various practical reasons <br>
21:13 <br>
which i'm not ready to discuss right now <br>
21:16 <br>
but uh <br>
21:20 <br>
but yeah so bonding curves are <br>
21:21 <br>
relatively new technology but we're not <br>
21:25 <br>
we're definitely not the first to try it <br>
21:27 <br>
okay maybe <br>
21:28 <br>
this will be my last question yes sorry <br>
21:31 <br>
i mean <br>
21:32 <br>
yes it's like the bonding curve we <br>
21:34 <br>
believe adds stability <br>
21:36 <br>
to the whole swarm ecosystem it also <br>
21:38 <br>
provides <br>
21:39 <br>
independent decentralized liquidity <br>
21:42 <br>
which also again strengthens <br>
21:44 <br>
uh swarm as a project and in a way <br>
21:48 <br>
uh you know it's uh we see a lot of <br>
21:51 <br>
projects in bedford <br>
21:53 <br>
so like curve dot finance well it's a <br>
21:56 <br>
curve <br>
21:57 <br>
you know uh the where they where they <br>
21:59 <br>
got the name from so there is a lot of <br>
22:01 <br>
innovation in this space <br>
22:03 <br>
um and uh i would say i would expect <br>
22:06 <br>
that we will see more and more projects <br>
22:08 <br>
uh in future also um doing <br>
22:12 <br>
similar approaches yeah i totally agree <br>
22:14 <br>
you can also already see it like in <br>
22:16 <br>
research and in like <br>
22:17 <br>
some even some github commits from other <br>
22:19 <br>
projects that like most of them are <br>
22:21 <br>
experimenting at least with with curves <br>
22:24 <br>
and i <br>
22:25 <br>
i am all for it one this is a very <br>
22:27 <br>
technical question <br>
22:28 <br>
danny from me to you why oh why <br>
22:31 <br>
um do we have 16 decimals like every erc <br>
22:35 <br>
20 compatible token has 18 decimals and <br>
22:37 <br>
are you <br>
22:38 <br>
are we trying <br>
22:42 <br>
and it has to do with the bonding curve <br>
22:45 <br>
so basically what we want to avoid <br>
22:47 <br>
is that in a situation where most of the <br>
22:49 <br>
bus tokens are sold back to the bonding <br>
22:52 <br>
curve <br>
22:53 <br>
we want to avoid the situation when some <br>
22:55 <br>
tokens are actually free <br>
22:57 <br>
when they cost zero and in order to <br>
23:00 <br>
avoid that <br>
23:02 <br>
in addition to the bank or function <br>
23:04 <br>
there's a little linear <br>
23:06 <br>
[Music] <br>
23:07 <br>
so basically we need to add the same <br>
23:10 <br>
amount of <br>
23:11 <br>
atomic units to the price <br>
23:15 <br>
in bus and in <br>
23:18 <br>
in die and we want that <br>
23:22 <br>
to be insignificant roughly one percent <br>
23:25 <br>
and one percent is two orders of two <br>
23:27 <br>
decimal orders of magnitude <br>
23:30 <br>
so it's a purely technical decision <br>
23:35 <br>
i hope it will not upset an upset <br>
23:39 <br>
anything <br>
23:39 <br>
no the power of 16 is still a very big <br>
23:43 <br>
number <br>
23:43 <br>
yes yes that's true um um <br>
23:46 <br>
no it won't upset people i think it's <br>
23:48 <br>
even a good thing because you can <br>
23:50 <br>
recognize all the scam tokens because <br>
23:51 <br>
they have 18 decimals <br>
23:53 <br>
and so this is you know only now maybe <br>
23:56 <br>
the scam tokens <br>
23:57 <br>
learns that we have 16 decimals so like <br>
24:00 <br>
if you see a pcz token with 16 decimals <br>
24:02 <br>
that doesn't mean <br>
24:04 <br>
that this one is failed right right you <br>
24:05 <br>
will see one with 18 <br>
24:07 <br>
that's a scam for sure yeah exactly yeah <br>
24:09 <br>
yeah <br>
24:10 <br>
exactly thank you rinky for the <br>
24:11 <br>
disclaimer um <br>
24:14 <br>
so i'll go back to my list of questions <br>
24:18 <br>
here thank you so much danny that's that <br>
24:20 <br>
was like a very thorough explanation i <br>
24:22 <br>
think <br>
24:23 <br>
um this is a this is one that <br>
24:26 <br>
probably for you has the over generous <br>
24:29 <br>
free time <br>
24:30 <br>
settlement issue being fixed i have no <br>
24:32 <br>
idea what this means but <br>
24:34 <br>
maybe some of you guys do <br>
24:37 <br>
um yeah so i'll provide a bit of <br>
24:40 <br>
background so this is about the <br>
24:42 <br>
um the ratio at which <br>
24:45 <br>
nodes give each other uh free bandwidth <br>
24:48 <br>
that is the <br>
24:49 <br>
the time refreshment ratio um it was set <br>
24:52 <br>
at a certain <br>
24:53 <br>
uh parameter and then we increased the <br>
24:56 <br>
third we interrupt but <br>
24:57 <br>
you have to explain this what does this <br>
24:59 <br>
mean what what does this mean even like <br>
25:01 <br>
notes give each other time <br>
25:03 <br>
that sounds very romantic but can you <br>
25:05 <br>
explain that <br>
25:07 <br>
yeah so this is the feature of <br>
25:09 <br>
time-based sentiments right <br>
25:10 <br>
um as you know before swarm nodes <br>
25:14 <br>
all gave each other bandwidth in <br>
25:16 <br>
exchange for money <br>
25:19 <br>
and we realized a couple of months ago <br>
25:21 <br>
actually that <br>
25:23 <br>
this kind of creates a critical <br>
25:25 <br>
dependency <br>
25:26 <br>
from the swarm network to the blockchain <br>
25:28 <br>
so if for whatever reason <br>
25:30 <br>
the blockchain wouldn't operate or <br>
25:34 <br>
nodes maybe would not like trust the <br>
25:36 <br>
blockchain anymore <br>
25:37 <br>
for security reasons or whatever then <br>
25:40 <br>
this whole swarm network would fall <br>
25:41 <br>
apart and we don't want it <br>
25:43 <br>
um and we figured out that <br>
25:48 <br>
we do want to keep these monetary <br>
25:50 <br>
settlements <br>
25:51 <br>
but alongside that we introduced a <br>
25:53 <br>
different kind of sediment layer which <br>
25:55 <br>
we call time-based settlements <br>
25:57 <br>
so the time-based settlements is <br>
25:59 <br>
basically nodes giving each other <br>
26:01 <br>
a certain kind of bandwidth capacity <br>
26:05 <br>
for free so they don't need to pay for <br>
26:07 <br>
that and and what you see then <br>
26:08 <br>
is that the two most important reasons <br>
26:10 <br>
for the swap bandwidth incentives <br>
26:13 <br>
is basically spam protection and <br>
26:16 <br>
um to incentivize the nodes collaborates <br>
26:19 <br>
and <br>
26:19 <br>
with time-based settlements they are <br>
26:22 <br>
preserved <br>
26:24 <br>
so the time based settlements is there <br>
26:26 <br>
if notes don't <br>
26:28 <br>
create like don't use a lot of bandwidth <br>
26:31 <br>
but if you would <br>
26:32 <br>
like start download the movie or <br>
26:34 <br>
whatever then you <br>
26:35 <br>
consume more than time based settlement <br>
26:37 <br>
and you use the monetary settlements <br>
26:40 <br>
um so in the p06 <br>
26:43 <br>
the time-based settlements basically <br>
26:45 <br>
nodes gave each other too much <br>
26:46 <br>
free bandwidth and this has been reduced <br>
26:49 <br>
now with the 1.0 release candidate <br>
26:51 <br>
and yeah we're going to run out today <br>
26:53 <br>
and and see how it behaves <br>
26:56 <br>
okay so the notes were too generous <br>
26:59 <br>
exactly okay cool so uh this is an <br>
27:02 <br>
interesting question i think <br>
27:04 <br>
what about the high gas prices on <br>
27:06 <br>
ethereum <br>
27:07 <br>
is an off chain or level 2 solution <br>
27:09 <br>
planned <br>
27:12 <br>
so let me answer that so we have <br>
27:15 <br>
researched <br>
27:18 <br>
the alternatives some layer two <br>
27:19 <br>
solutions and <br>
27:21 <br>
basically we settled on <br>
27:24 <br>
a side chain called hexadiene <br>
27:27 <br>
x-die is a evm-based side chain which <br>
27:31 <br>
of ethereum meaning that it has this <br>
27:34 <br>
exact same smart contract mechanics as <br>
27:36 <br>
ethereum itself <br>
27:38 <br>
uh same transaction format <br>
27:41 <br>
same kind of apis which means <br>
27:45 <br>
that the transition is relatively <br>
27:47 <br>
painless <br>
27:49 <br>
it has been operating for a while so it <br>
27:51 <br>
has a good track record <br>
27:52 <br>
it is used by other projects so <br>
27:56 <br>
so there's already some community trust <br>
27:59 <br>
around it <br>
28:00 <br>
uh it is cheaper and faster than <br>
28:03 <br>
ethereum may not and it can carry the <br>
28:06 <br>
kind of microtransactions <br>
28:07 <br>
that swarm requires but it's not a long <br>
28:10 <br>
term solution <br>
28:11 <br>
so this is a stop gap measure because <br>
28:14 <br>
basically ethereum scaling problems <br>
28:17 <br>
didn't catch up with <br>
28:19 <br>
one's requirements yet but we're really <br>
28:22 <br>
hopeful that <br>
28:24 <br>
there will be some standard scalability <br>
28:26 <br>
solution for ethereum <br>
28:28 <br>
maybe ethereum two maybe some roll up <br>
28:31 <br>
we don't know yet but when it comes <br>
28:34 <br>
around we can <br>
28:35 <br>
we can easily change that so x-die <br>
28:39 <br>
is a fully bridged uh side chain <br>
28:42 <br>
which means that bus tokens can move <br>
28:45 <br>
from the main to the side chain and back <br>
28:48 <br>
trustlessly and and safely <br>
28:52 <br>
and we're only committed to using the <br>
28:55 <br>
bus token <br>
28:56 <br>
but where these bus tokens live is <br>
29:00 <br>
it it can change depending on how <br>
29:01 <br>
technology develops <br>
29:03 <br>
and maybe for example if swarm's <br>
29:05 <br>
popularity explodes <br>
29:07 <br>
maybe it will be the very reason why xdi <br>
29:10 <br>
network will be clogged up <br>
29:12 <br>
so it's not an ultimate scaling solution <br>
29:15 <br>
but we think that for the <br>
29:17 <br>
time being which means months and maybe <br>
29:20 <br>
even years <br>
29:22 <br>
from now it's an adequate solution so <br>
29:25 <br>
maybe i want to add like one <br>
29:26 <br>
clarification you say that xdi <br>
29:28 <br>
people can move their tokens trustworthy <br>
29:31 <br>
i don't think that's correct <br>
29:33 <br>
because we we still depend on the bridge <br>
29:35 <br>
operators to <br>
29:36 <br>
uh to migrate the tokens so there is a <br>
29:39 <br>
certain trust in the operators of the <br>
29:41 <br>
bridge <br>
29:42 <br>
yes but they are in something else they <br>
29:44 <br>
are incentivized <br>
29:46 <br>
yeah yeah but um it is <br>
29:49 <br>
trustless to a certain degree but not as <br>
29:51 <br>
trustless as the imperial mainland <br>
29:53 <br>
rights <br>
29:54 <br>
exactly but yes but at least we <br>
29:57 <br>
as a team say we trusted so in that way <br>
30:00 <br>
you can trust it <br>
30:04 <br>
right yes okay um <br>
30:07 <br>
okay wow what is this what is the <br>
30:10 <br>
minimum <br>
30:10 <br>
expected storage performance for b <br>
30:14 <br>
is it a good idea to use nfs <br>
30:17 <br>
or distributed block storage to store <br>
30:20 <br>
chunks of multiple instances <br>
30:23 <br>
are are nvme ssds the best overall <br>
30:26 <br>
option does anyone know you know all <br>
30:28 <br>
these terms <br>
30:30 <br>
probably you guys should know this right <br>
30:31 <br>
no definitely this <br>
30:33 <br>
is the best option <br>
30:36 <br>
but the first one i think you mentioned <br>
30:38 <br>
nfs <br>
30:40 <br>
yeah nfs are distributed block storage <br>
30:42 <br>
like menio <br>
30:46 <br>
yeah we didn't test it with <br>
30:50 <br>
menio it works with <br>
30:53 <br>
ebs that's a <br>
30:56 <br>
on on aws but yeah minio and <br>
31:00 <br>
nfs really natural depending on the <br>
31:03 <br>
network <br>
31:06 <br>
speed on the disk speed again yeah with <br>
31:09 <br>
nfs servers so yeah <br>
31:11 <br>
but yeah faster the better but <br>
31:15 <br>
we didn't tested it as rinka said <br>
31:18 <br>
yeah it would be nice also community <br>
31:22 <br>
yeah just test it and yeah wrap your <br>
31:24 <br>
results in the node operator <br>
31:26 <br>
yeah yeah thank you so much for that <br>
31:29 <br>
it's it's fun to be able to trust on <br>
31:31 <br>
your knowledge because i had no idea <br>
31:33 <br>
what that all means <br>
31:34 <br>
um are there limits to the storage space <br>
31:38 <br>
a single b <br>
31:39 <br>
can take up the db capacity option was <br>
31:41 <br>
removed in 0.6.0 <br>
31:44 <br>
is there another way <br>
31:47 <br>
yeah thank you okay so okay so <br>
31:51 <br>
so the the this capacity was removed <br>
31:54 <br>
because <br>
31:55 <br>
because it's no longer it's no longer <br>
31:58 <br>
one <br>
31:59 <br>
parameter that that <br>
32:02 <br>
that is that is used but there's a <br>
32:05 <br>
reserve and the cache <br>
32:06 <br>
and the reserve is is a is a system-wide <br>
32:09 <br>
constant it's it's supposed to be the <br>
32:11 <br>
same <br>
32:12 <br>
reserve storage size for each each and <br>
32:15 <br>
every node so in that sense it's not <br>
32:16 <br>
configurable that's <br>
32:18 <br>
that's that's it's actually let's say <br>
32:21 <br>
let's see that's a part of the <br>
32:22 <br>
requirement of running a node that you <br>
32:24 <br>
that you have that space <br>
32:26 <br>
and as for the cache the cache itself is <br>
32:29 <br>
is is is configurable and and it can be <br>
32:32 <br>
bigger <br>
32:33 <br>
however however it's it it <br>
32:37 <br>
it's depending depending on the <br>
32:39 <br>
depending on the popular <br>
32:40 <br>
population of the of the of the network <br>
32:43 <br>
and and the number of uh chunks in it <br>
32:46 <br>
the number of the volume of data in it <br>
32:49 <br>
is <br>
32:50 <br>
it is it's going to be after after first <br>
32:53 <br>
certain size <br>
32:54 <br>
it's not going to be more profitable to <br>
32:56 <br>
to operate a bigger bigger cache <br>
32:58 <br>
uh most most likely so so there's <br>
33:02 <br>
this practically there's there's no gain <br>
33:04 <br>
in having <br>
33:05 <br>
having a lot of storage for one node <br>
33:08 <br>
and and i hope it answers the question <br>
33:11 <br>
and <br>
33:12 <br>
and just just landed the side notes here <br>
33:14 <br>
so there's there's in this form there's <br>
33:15 <br>
such a thing as pinning <br>
33:17 <br>
that certain certain content that you <br>
33:19 <br>
can you can <br>
33:20 <br>
pin to your local node and and and have <br>
33:23 <br>
it <br>
33:24 <br>
as part of your part of your um <br>
33:28 <br>
storage that is not removed by the by <br>
33:30 <br>
the by the garbage collection processes <br>
33:34 <br>
but this is important to know that this <br>
33:36 <br>
pin content <br>
33:37 <br>
is is uh calculated on top of <br>
33:40 <br>
the other requirements so if you want to <br>
33:42 <br>
pin content like lots of <br>
33:44 <br>
movies you have to take care that the <br>
33:46 <br>
space for those pinned content <br>
33:48 <br>
is available unless unless the pin <br>
33:50 <br>
content is otherwise protected <br>
33:52 <br>
in which case it's part of the the <br>
33:53 <br>
reserve anyway so so once again if you <br>
33:56 <br>
have postage stamp <br>
33:58 <br>
if you if you have if you have content <br>
34:00 <br>
which you want to pin <br>
34:01 <br>
with expired postage stamps you need <br>
34:04 <br>
you need to take care that that the the <br>
34:07 <br>
storage <br>
34:08 <br>
space that it takes up is is extra and <br>
34:10 <br>
provided over over on top of the the <br>
34:12 <br>
questions <br>
34:14 <br>
i hope it's clear that totally makes <br>
34:16 <br>
sense to me <br>
34:17 <br>
but i did read the book of swarm um <br>
34:20 <br>
this is i want to add like a small <br>
34:21 <br>
clarification here <br>
34:23 <br>
if i may so the reserve <br>
34:26 <br>
is for those chunks that you're supposed <br>
34:28 <br>
to store <br>
34:29 <br>
based on like your proximity to the <br>
34:31 <br>
chunks so that's what we call the <br>
34:32 <br>
natural location in the white paper <br>
34:34 <br>
so all the chunks that you are like <br>
34:36 <br>
storing based because they are so close <br>
34:37 <br>
to you <br>
34:38 <br>
they go to the reserve and apart from <br>
34:41 <br>
that note operators also <br>
34:43 <br>
what we call cash chunks whenever i am <br>
34:46 <br>
forwarding a chunk to his destination <br>
34:48 <br>
and this caching is actually <br>
34:50 <br>
incentivized because if i if i'm <br>
34:52 <br>
cashing a chunk and i get requested it <br>
34:55 <br>
means that i don't need to pay <br>
34:56 <br>
to get it to my drone stream here but i <br>
34:58 <br>
can immediately set it and have pure <br>
34:59 <br>
profit <br>
35:00 <br>
and this is the cash right so that's why <br>
35:03 <br>
the cash is modifiable <br>
35:05 <br>
like it doesn't really matter if you <br>
35:06 <br>
cash or not like the network will <br>
35:08 <br>
still function if no note caches that's <br>
35:10 <br>
why it doesn't matter <br>
35:12 <br>
but the reserve capacity really should <br>
35:14 <br>
stay the same <br>
35:15 <br>
because this is yeah this is what you're <br>
35:17 <br>
supposed to store based on your location <br>
35:20 <br>
okay let's let's uh go on what is a <br>
35:23 <br>
non-mineable <br>
35:24 <br>
overlay and how exactly is it used <br>
35:28 <br>
again i have no idea what what they mean <br>
35:30 <br>
by that yeah <br>
35:31 <br>
so i think i can answer that um whoever <br>
35:33 <br>
asked this question is is keeping a very <br>
35:35 <br>
close eye <br>
35:36 <br>
to our pr so this vr is out there i <br>
35:39 <br>
think maybe it's merged right now <br>
35:40 <br>
maybe it will be merged in the upcoming <br>
35:43 <br>
hour but <br>
35:44 <br>
um basically swarm has a requirement <br>
35:48 <br>
and we are always reasoning about swarm <br>
35:50 <br>
that node operators kind of like <br>
35:52 <br>
randomly choose <br>
35:53 <br>
their overlay autos right and then we <br>
35:56 <br>
know that all artists <br>
35:57 <br>
are good sorry an overlay address is the <br>
36:00 <br>
address like the ip address of the note <br>
36:02 <br>
no it's it's the um it's the address <br>
36:05 <br>
that basically defines what chunks <br>
36:06 <br>
you'll be storing <br>
36:08 <br>
okay it's similar to the ethereum <br>
36:10 <br>
address <br>
36:12 <br>
yeah so <br>
36:15 <br>
we're always assuming that the overlay <br>
36:18 <br>
artists are <br>
36:19 <br>
evenly distributed around the network <br>
36:22 <br>
however um the overlay artist before was <br>
36:25 <br>
purely <br>
36:26 <br>
based on the ethereum honors of the user <br>
36:30 <br>
so by choosing like by kind of like what <br>
36:32 <br>
you call mining <br>
36:33 <br>
uh trying out many private keys i could <br>
36:36 <br>
try out like so many times <br>
36:38 <br>
until i have a particular overlay <br>
36:40 <br>
address that would land into a certain <br>
36:41 <br>
neighborhood <br>
36:42 <br>
now this opens all kinds of bad <br>
36:45 <br>
scenarios maybe the very first one <br>
36:47 <br>
just that the overlay offices are not <br>
36:48 <br>
evenly distributed so one space of the <br>
36:50 <br>
network has more capacity than another <br>
36:52 <br>
space as <br>
36:53 <br>
network but one could also think about <br>
36:55 <br>
more kind of adversarial scenarios where <br>
36:58 <br>
i would just <br>
36:58 <br>
kind of mine thousands of others in one <br>
37:01 <br>
neighborhood and thereby kind of <br>
37:03 <br>
completely <br>
37:04 <br>
isolating that neighborhood from the <br>
37:05 <br>
network <br>
37:07 <br>
so this is where the non-mineable <br>
37:09 <br>
overlay obviously <br>
37:10 <br>
comes in so basically we added one <br>
37:12 <br>
additional ingredients <br>
37:13 <br>
um what defines what your overlay iris <br>
37:16 <br>
is going to be <br>
37:17 <br>
and this is the block hash at which your <br>
37:20 <br>
notes <br>
37:20 <br>
registered so basically now in order to <br>
37:24 <br>
choose a neighborhoods i need to do a <br>
37:27 <br>
transaction <br>
37:28 <br>
on the die test chain so this will cost <br>
37:30 <br>
real money <br>
37:32 <br>
so node operators can still choose kind <br>
37:35 <br>
of <br>
37:35 <br>
what not the diet test chain right you <br>
37:37 <br>
don't need to pay for that <br>
37:39 <br>
yeah so you say like you have to do a <br>
37:40 <br>
transaction on the diet test chain what <br>
37:42 <br>
what that what do you mean by that <br>
37:45 <br>
side chain chain okay just i'm just <br>
37:47 <br>
correcting you because you know <br>
37:49 <br>
you know these details matter um okay <br>
37:52 <br>
this is uh to me it's at least clear i <br>
37:54 <br>
hope also to the <br>
37:55 <br>
author of that question um not operators <br>
37:58 <br>
nothing changes like they don't <br>
38:00 <br>
maybe the only thing they notice is that <br>
38:01 <br>
the overlay artist is different <br>
38:04 <br>
but apart from that this is really you <br>
38:06 <br>
know something <br>
38:07 <br>
internal under the hood um the next <br>
38:10 <br>
question <br>
38:11 <br>
can the swap initial deposit be adjusted <br>
38:14 <br>
and if so what is the allowed range this <br>
38:16 <br>
was <br>
38:17 <br>
of course a question um you have to <br>
38:19 <br>
explain this because <br>
38:21 <br>
especially i think from china we get a <br>
38:23 <br>
lot of questions about <br>
38:25 <br>
um staking mining you know all that kind <br>
38:28 <br>
of questions where we actually <br>
38:29 <br>
see like this is just not true um <br>
38:32 <br>
so if we tell people like you need to <br>
38:35 <br>
put in <br>
38:36 <br>
some bus right you know in order to <br>
38:39 <br>
operate your node <br>
38:40 <br>
is this true is this not you know can <br>
38:42 <br>
you can you talk about that because this <br>
38:44 <br>
is this is <br>
38:44 <br>
this is something that was very <br>
38:45 <br>
confusing no the answer is a resounding <br>
38:49 <br>
oh some of you have to mute yeah <br>
38:53 <br>
so the answer is guys please <br>
38:57 <br>
oh god <br>
39:00 <br>
okay try again so the answer is a <br>
39:03 <br>
resounding no <br>
39:05 <br>
you don't need to state buzz in order to <br>
39:07 <br>
operate a node you can start with zero <br>
39:09 <br>
and you can earn your way <br>
39:12 <br>
into the network by providing the <br>
39:14 <br>
service <br>
39:16 <br>
i'm gonna double check this is this this <br>
39:18 <br>
this is so i <br>
39:20 <br>
i'm gonna restate this right i can start <br>
39:23 <br>
a b note <br>
39:24 <br>
i don't have to put in any bus i don't <br>
39:26 <br>
have to buy bus <br>
39:27 <br>
i can just run it and earn bus with it <br>
39:33 <br>
um a small side note yes yes with zero <br>
39:38 <br>
in theory you can earn your money to get <br>
39:41 <br>
in and actually all the smart contracts <br>
39:43 <br>
are completely prepared <br>
39:44 <br>
uh that instead of note operators paying <br>
39:47 <br>
you uh node operators can pay you by <br>
39:50 <br>
deploying a checkbook for you <br>
39:52 <br>
however this functionality needs a bit <br>
39:54 <br>
more wiring in and we'll <br>
39:55 <br>
we're planning um like a non-blackboard <br>
39:57 <br>
compatible update <br>
39:58 <br>
that does this at a later point but <br>
40:02 <br>
the requirement to join the swarm with <br>
40:05 <br>
zero bzz is perfectly possible and i <br>
40:08 <br>
know michelle we have been having a <br>
40:10 <br>
discussion about this like two months <br>
40:11 <br>
ago <br>
40:12 <br>
but this now is very easy especially <br>
40:14 <br>
because of the time based settlements <br>
40:15 <br>
that we spoke about before <br>
40:17 <br>
so this is no no problem anymore and uh <br>
40:20 <br>
yeah okay <br>
40:20 <br>
so i'm gonna ask this once once again <br>
40:23 <br>
just to have it <br>
40:24 <br>
clear so i can start a b note and start <br>
40:27 <br>
earning bcc without putting in any bzz <br>
40:32 <br>
now today i will on the on june 21st <br>
40:35 <br>
about the earning bcc i'm not too sure <br>
40:38 <br>
because <br>
40:38 <br>
it might be but i don't know if note <br>
40:40 <br>
operators are <br>
40:42 <br>
paying always to a checkbook or if they <br>
40:44 <br>
can also pay <br>
40:45 <br>
to an ethereum address so like <br>
40:48 <br>
anyways if this is not enabled right now <br>
40:51 <br>
it will be like <br>
40:52 <br>
super easy to enable at a later point um <br>
40:55 <br>
yeah maybe yeah <br>
40:58 <br>
yeah gregor thank you um there was a <br>
41:01 <br>
there was one i would also <br>
41:02 <br>
ask here a question because a lot of <br>
41:04 <br>
times i get another <br>
41:06 <br>
question do does a node operator need to <br>
41:08 <br>
deposit either <br>
41:10 <br>
or x die in order to run the node <br>
41:13 <br>
um the node operator um i mean like what <br>
41:17 <br>
i said right you can be a node operator <br>
41:19 <br>
without deploying a checkbook so in that <br>
41:21 <br>
sense <br>
41:22 <br>
no x die is needed as well however xl is <br>
41:26 <br>
an <br>
41:26 <br>
evm chain and as an evm chain you will <br>
41:29 <br>
need <br>
41:30 <br>
you will need guess the gas costs on <br>
41:34 <br>
x die are significantly lower than the <br>
41:37 <br>
guest cost on the ethereum main chain <br>
41:39 <br>
and i even believe <br>
41:40 <br>
that the xdi team is operating a faucet <br>
41:44 <br>
for getting x die for your transaction <br>
41:47 <br>
costs <br>
41:50 <br>
so if i understand it correctly it's <br>
41:52 <br>
more practical <br>
41:54 <br>
and easier if there's like a little bit <br>
41:56 <br>
of exercise <br>
41:58 <br>
on the account yes yes yeah if you want <br>
42:01 <br>
to deploy your checkbook if you want to <br>
42:02 <br>
cash out your checks <br>
42:04 <br>
like all these operations if you want to <br>
42:05 <br>
do an operation on the blockchain <br>
42:08 <br>
with your checkbooks with your caches <br>
42:09 <br>
with your like <br>
42:11 <br>
uh buying a postage stamp you will need <br>
42:13 <br>
xdi <br>
42:14 <br>
alongside vcc like a small amount to pay <br>
42:16 <br>
for the guest piece <br>
42:20 <br>
thank you thank you um this this is this <br>
42:23 <br>
is i hope this sheds clarity because <br>
42:25 <br>
there were so many <br>
42:27 <br>
you know confused people asking about <br>
42:29 <br>
speaking and you know <br>
42:30 <br>
there are a lot of stories going around <br>
42:33 <br>
and maybe i can promise an answer <br>
42:35 <br>
maybe i can promise an answer for the <br>
42:37 <br>
question whether not operators will be <br>
42:39 <br>
able to earn bcc <br>
42:40 <br>
without even having bcc i can promise <br>
42:42 <br>
this answer like i need to check this <br>
42:45 <br>
okay no problem i'll ask you again <br>
42:46 <br>
monday at the live event <br>
42:48 <br>
so yes please greg or go ahead <br>
42:54 <br>
um maybe maybe it's a good thing <br>
42:58 <br>
there are inferiors uh also mentioned <br>
43:02 <br>
and uh for that there will be uh <br>
43:05 <br>
needed speaking right and this maybe is <br>
43:08 <br>
for those who read the book of swan can <br>
43:10 <br>
also be <br>
43:11 <br>
the source of confusion maybe maybe <br>
43:13 <br>
victor you can <br>
43:14 <br>
shed more light on this okay so <br>
43:18 <br>
so indeed indeed indeed there's various <br>
43:20 <br>
layers of <br>
43:22 <br>
of of potential um <br>
43:26 <br>
incentivization for for persistence and <br>
43:28 <br>
persisting files <br>
43:29 <br>
so the current version of swarm is only <br>
43:32 <br>
going to implement <br>
43:36 <br>
indirect positive rewards and <br>
43:39 <br>
incentivization <br>
43:40 <br>
this this is this means that that uh <br>
43:43 <br>
those that that it's it's it's the <br>
43:46 <br>
whole system has an emergent incentive <br>
43:49 <br>
to to preserve <br>
43:50 <br>
to preserve the files because he was <br>
43:52 <br>
doing so you <br>
43:54 <br>
you you comply if you comply with the <br>
43:56 <br>
protocol then you're basically <br>
43:58 <br>
serving serving the the your interest as <br>
44:01 <br>
a bus token holder <br>
44:02 <br>
and and uh and and instantly works <br>
44:06 <br>
positively like that but the the <br>
44:09 <br>
staking solution is is kind of is <br>
44:11 <br>
usually under <br>
44:12 <br>
uh so under under the under the heading <br>
44:15 <br>
of <br>
44:16 <br>
negative incentives which means that you <br>
44:18 <br>
you <br>
44:19 <br>
you have a mechanism that <br>
44:20 <br>
disincentivizes you <br>
44:23 <br>
to lose the contents prevents you from <br>
44:25 <br>
from losing the content <br>
44:26 <br>
by by the the threat of threat of some <br>
44:30 <br>
some sort of confiscation or stress of <br>
44:33 <br>
of of having you having your stay <br>
44:36 <br>
forfeited <br>
44:37 <br>
and and and that this this this layer of <br>
44:40 <br>
insurance is not part of the current <br>
44:43 <br>
release <br>
44:44 <br>
but of course part of the part of the <br>
44:46 <br>
release plan and then the roadmap <br>
44:48 <br>
later and there's gonna be various <br>
44:49 <br>
solutions for this <br>
44:51 <br>
uh in the in the in the in the <br>
44:55 <br>
in the in the in the middle of these two <br>
44:57 <br>
solutions there's a <br>
44:58 <br>
there's a there's a there's a <br>
45:02 <br>
little solution which is which is the <br>
45:04 <br>
direct positive rewards <br>
45:05 <br>
which is a system which uh directly <br>
45:08 <br>
redistributes uh some of the postage <br>
45:11 <br>
stamp revenue so <br>
45:12 <br>
the revenue that that is generated by by <br>
45:15 <br>
people paying in <br>
45:16 <br>
a kind of <br>
45:20 <br>
rant-like <br>
45:24 <br>
post for for that for the for storing <br>
45:25 <br>
their files and in the same time <br>
45:27 <br>
solution you <br>
45:28 <br>
know in this institution <br>
45:31 <br>
you you will get this this revenue <br>
45:34 <br>
redistributed among <br>
45:35 <br>
the the stores that that <br>
45:38 <br>
that are entitled to it based on the <br>
45:41 <br>
less <br>
45:42 <br>
storage business storage activity so <br>
45:46 <br>
this these are going to be the upgrades <br>
45:48 <br>
on a road map <br>
45:49 <br>
and and <br>
45:52 <br>
so once again to wrap up there's no <br>
45:54 <br>
staking currently needed <br>
45:56 <br>
on the on the part of the boards uh <br>
46:00 <br>
even the the the the amount that you <br>
46:03 <br>
need to have is is is only to cover the <br>
46:07 <br>
the cost of of having one transaction uh <br>
46:10 <br>
attached to your <br>
46:11 <br>
to your to the rest that that that is <br>
46:15 <br>
the basis <br>
46:15 <br>
for you for your swarm ideas for you <br>
46:17 <br>
really this was the <br>
46:19 <br>
minor overlay that that ring i mentioned <br>
46:22 <br>
okay and and that's basically it <br>
46:26 <br>
thank you for the for the for the <br>
46:28 <br>
wrap-up <br>
46:29 <br>
um this is um so guys we're gonna now <br>
46:32 <br>
have like a fun round <br>
46:36 <br>
um so imagine that you're in a tv show <br>
46:38 <br>
now <br>
46:39 <br>
um prometheus metrics that b exposes <br>
46:42 <br>
are a bit obscure according to one <br>
46:45 <br>
author <br>
46:46 <br>
so this this uh this question oscar <br>
46:49 <br>
is basically asking us to explain in <br>
46:52 <br>
layman's terms <br>
46:53 <br>
some terms that we use in the prometheus <br>
46:56 <br>
um <br>
46:57 <br>
interface are in what is it like the <br>
46:58 <br>
logs or the debug interface <br>
47:00 <br>
so um you can only answer if you can do <br>
47:03 <br>
it in one sentence that's the rule of <br>
47:05 <br>
the game okay so i'm gonna just say a <br>
47:07 <br>
term <br>
47:08 <br>
and then one of you has to answer it but <br>
47:10 <br>
in one sentence <br>
47:11 <br>
and with one sentence i mean like 15 <br>
47:13 <br>
seconds not like <br>
47:14 <br>
not like a sentence that takes on and on <br>
47:16 <br>
and on and on <br>
47:17 <br>
okay get ready for this <br>
47:22 <br>
accounting <br>
47:24 <br>
i just want to say that the prometheus <br>
47:26 <br>
metrics so far <br>
47:28 <br>
has been like you know the users the <br>
47:31 <br>
intended users of those are the <br>
47:33 <br>
developers <br>
47:33 <br>
um however of course they can also be <br>
47:36 <br>
used by node operators um <br>
47:38 <br>
but the prometheus metrics haven't been <br>
47:41 <br>
kind of <br>
47:41 <br>
sanitized to be useful for node for node <br>
47:45 <br>
operators so maybe <br>
47:46 <br>
does this mean you can't you don't want <br>
47:48 <br>
to answer the question <br>
47:49 <br>
or you don't <br>
47:56 <br>
so i'll just i'll just but you have to <br>
47:58 <br>
do it in one sentence <br>
48:00 <br>
accounting accounting is there is the <br>
48:03 <br>
peer-to-peer tracking of <br>
48:07 <br>
bandwidth use between peers <br>
48:10 <br>
yes that's a great sentence batch store <br>
48:22 <br>
is the place where information about <br>
48:25 <br>
uh about postage patty matches <br>
48:29 <br>
are synchronized with the blockchain <br>
48:32 <br>
okay <br>
48:32 <br>
local store this is the store for chunks <br>
48:37 <br>
and <br>
48:37 <br>
one related information full storage <br>
48:43 <br>
full storage yes <br>
48:53 <br>
[Music] <br>
49:01 <br>
the synchronization of chunks in within <br>
49:04 <br>
the neighborhood <br>
49:06 <br>
next one pull sync that's the <br>
49:09 <br>
synchronization of shanks in the <br>
49:10 <br>
neighborhood <br>
49:12 <br>
for sure this is the component that <br>
49:17 <br>
that that is running in the background <br>
49:20 <br>
to make sure that chances that you <br>
49:22 <br>
uploaded on your local road <br>
49:24 <br>
reach the network and ultimately are <br>
49:26 <br>
pushed to the <br>
49:27 <br>
natural location <br>
49:37 <br>
why didn't we call it just upload them <br>
49:39 <br>
anyway that was not a question <br>
49:42 <br>
retrieval download <br>
49:48 <br>
swap <br>
49:51 <br>
swap is the components that does the <br>
49:55 <br>
peer-to-peer accounting to incentivize <br>
49:57 <br>
for <br>
49:57 <br>
for the bandwidth uh bandwidth account <br>
50:00 <br>
okay <br>
50:01 <br>
i that that was it thank you so much for <br>
50:03 <br>
this first round <br>
50:04 <br>
um i'm i'm so sorry that the floss <br>
50:07 <br>
capacitor is not <br>
50:08 <br>
in b anymore i was i was looking for <br>
50:11 <br>
that because that was my favorite term <br>
50:13 <br>
ever in in storm um but it's gone <br>
50:17 <br>
so i don't know if i if i can do like a <br>
50:20 <br>
request <br>
50:20 <br>
but i think i'm gonna make an issue on <br>
50:22 <br>
github like asking like can we just have <br>
50:23 <br>
this back <br>
50:24 <br>
because it's so funky anyway um <br>
50:28 <br>
how much time do we have left uh casper <br>
50:31 <br>
do i have to do we have to be like very <br>
50:33 <br>
strict with time <br>
50:34 <br>
or do we have more time <br>
50:40 <br>
no we got maybe gasper already ran away <br>
50:44 <br>
well anyway let's go to the next <br>
50:46 <br>
question um <br>
50:49 <br>
okay b seems to shred through <br>
50:52 <br>
not boxes because of its high connection <br>
50:55 <br>
count i <br>
50:56 <br>
i had this is this likely to change so <br>
50:59 <br>
so i was running so i'm running like b <br>
51:02 <br>
notes on that note right <br>
51:03 <br>
and when i just connect them in my home <br>
51:05 <br>
network um <br>
51:07 <br>
and i and i also then um watch a movie <br>
51:10 <br>
for instance or i do <br>
51:11 <br>
or anything my kids start i don't know <br>
51:13 <br>
playing minecraft then the whole network <br>
51:15 <br>
just crashes <br>
51:16 <br>
like all the routers in the house just <br>
51:17 <br>
say like this i'm gonna reboot <br>
51:19 <br>
like so it's why <br>
51:22 <br>
is this and will it be fixed <br>
51:27 <br>
because of its high connection count is <br>
51:29 <br>
there like is there like a limit on the <br>
51:31 <br>
connection count <br>
51:32 <br>
basically my isp is saying like what <br>
51:34 <br>
you're doing is not <br>
51:36 <br>
okay you're a hacker or something and <br>
51:38 <br>
then they shut down my <br>
51:42 <br>
incoming yeah i think i can answer it um <br>
51:45 <br>
[Music] <br>
51:47 <br>
b requires a certain number of <br>
51:49 <br>
connection <br>
51:50 <br>
per bin and a bin is like a term <br>
51:52 <br>
specific for codemglia <br>
51:54 <br>
i think right now it is eight <br>
51:56 <br>
connections per bin <br>
51:58 <br>
and every time the network doubles we <br>
52:00 <br>
have another bin so i think <br>
52:01 <br>
right now we have like 15 depths so that <br>
52:04 <br>
means that there are two to the power of <br>
52:06 <br>
15 <br>
52:07 <br>
uh nodes in the network is it correctly <br>
52:08 <br>
victor <br>
52:10 <br>
yes like and every time the network <br>
52:14 <br>
doubles <br>
52:14 <br>
your node would basically have eight <br>
52:16 <br>
more connections to maintain <br>
52:19 <br>
um now there are so certain like a <br>
52:22 <br>
couple of edge conditions that nodes <br>
52:24 <br>
would <br>
52:24 <br>
connect to more than eight spears per <br>
52:27 <br>
bin <br>
52:28 <br>
and i would expect this to be sanitized <br>
52:30 <br>
in the future that the node <br>
52:32 <br>
is just maintaining those in in a better <br>
52:34 <br>
way <br>
52:36 <br>
um if you're really having issues with <br>
52:39 <br>
that you can <br>
52:40 <br>
manually manage and maintain all your <br>
52:42 <br>
connections so there are debug api <br>
52:44 <br>
endpoints where you can just say <br>
52:46 <br>
disconnect to node so <br>
52:48 <br>
to all the node operators that have <br>
52:50 <br>
issues with that and are serious about <br>
52:51 <br>
maintaining and operating your nodes if <br>
52:53 <br>
they see <br>
52:54 <br>
more than eight nodes in a particular <br>
52:55 <br>
bin you can just <br>
52:57 <br>
kick out any notes above eight <br>
53:01 <br>
okay so the next question i i know it's <br>
53:05 <br>
it's meant as a yes it's meant as a <br>
53:06 <br>
technical question but i want to ask <br>
53:08 <br>
this to gregor <br>
53:09 <br>
um maybe just add the previous one uh <br>
53:14 <br>
from what is discussed with some other <br>
53:16 <br>
uh people <br>
53:18 <br>
does maybe your question relates to the <br>
53:20 <br>
initial boot up <br>
53:21 <br>
of the b note when when it's more <br>
53:24 <br>
actively syncing with the network <br>
53:27 <br>
maybe that's the case yeah so so <br>
53:29 <br>
basically <br>
53:30 <br>
then there would just need to be some <br>
53:32 <br>
throttling on the initial <br>
53:35 <br>
foot <br>
53:38 <br>
yeah because uh when it's like fully <br>
53:40 <br>
synced it all goes down so <br>
53:42 <br>
anyway gregor good that you're talking <br>
53:44 <br>
because i wanted to ask this question to <br>
53:46 <br>
you specifically <br>
53:47 <br>
um because of your cypherpunk background <br>
53:51 <br>
um so someone is asking can i run uh <br>
53:56 <br>
no a mainnet b note on and then it's <br>
53:58 <br>
just like insert like aws or like uh <br>
54:01 <br>
clouds what are they called like all <br>
54:03 <br>
these cloud providers right <br>
54:05 <br>
and so i know technically we already <br>
54:07 <br>
explained you can run a mainnet note on <br>
54:10 <br>
aws <br>
54:11 <br>
but i want you to explain why this is <br>
54:13 <br>
not <br>
54:14 <br>
i mean if you really believe in what <br>
54:15 <br>
swarm is trying to achieve why this is <br>
54:17 <br>
maybe not the best idea <br>
54:19 <br>
i mean from an ideological standpoint <br>
54:22 <br>
yes of course <br>
54:22 <br>
it's you know uh i guess it starts just <br>
54:25 <br>
with this <br>
54:26 <br>
simple phrase we all know my uh my clip <br>
54:29 <br>
to my keys <br>
54:30 <br>
you know and it also applies the same <br>
54:32 <br>
logic uh <br>
54:33 <br>
the same cycle path logic if you want to <br>
54:35 <br>
say uh if you want to say so it's like <br>
54:37 <br>
basically <br>
54:38 <br>
the user repairing the control having <br>
54:40 <br>
the agency so <br>
54:42 <br>
by uh hosting b nodes uh on <br>
54:45 <br>
cloud providers we kind of put <br>
54:49 <br>
this control into third party scans so <br>
54:52 <br>
well it might just work perfectly <br>
54:56 <br>
it does not really add to the <br>
55:00 <br>
the whole idea of decentralization in <br>
55:02 <br>
this case you've seen <br>
55:03 <br>
especially already this year really <br>
55:06 <br>
amazing <br>
55:07 <br>
like really surprising moves uh <br>
55:10 <br>
uh how amazon was uh overnight cutting <br>
55:14 <br>
off <br>
55:14 <br>
uh some um ad providers and developers <br>
55:17 <br>
and <br>
55:18 <br>
things like that so uh of course we <br>
55:21 <br>
would as <br>
55:21 <br>
a small foundation of support and <br>
55:24 <br>
endorse that people run <br>
55:26 <br>
be nodes on their own hardware this is <br>
55:28 <br>
also <br>
55:29 <br>
i think that node is actually written <br>
55:32 <br>
extensive on this topic because that not <br>
55:34 <br>
this <br>
55:36 <br>
issue yes i i always keep sending people <br>
55:40 <br>
to <br>
55:40 <br>
you know because people are often <br>
55:43 <br>
looking at aws for instance for like <br>
55:45 <br>
convenience <br>
55:46 <br>
and i and i try to then show them uh <br>
55:48 <br>
that with that note <br>
55:49 <br>
it's also very convenient and you stay <br>
55:52 <br>
within <br>
55:53 <br>
like the you know the goal and the <br>
55:55 <br>
vision of what swarm is trying to <br>
55:57 <br>
achieve <br>
55:59 <br>
um if others want to elaborate <br>
56:02 <br>
sure yeah please do i think this is an <br>
56:04 <br>
important it's not it's not a very <br>
56:06 <br>
technical topic but i think in a way <br>
56:08 <br>
our ideological philosophy behind swarm <br>
56:10 <br>
is also very technical <br>
56:12 <br>
um not for developers but like for <br>
56:14 <br>
people into philosophy and ideology it's <br>
56:16 <br>
it's quite <br>
56:17 <br>
you know it's quite a deep dive also i <br>
56:20 <br>
would have done it practically <br>
56:23 <br>
so if you take this idea of <br>
56:24 <br>
decentralization to the extreme <br>
56:26 <br>
then then amazon might not be really the <br>
56:29 <br>
most part of the solution <br>
56:32 <br>
the whole point of swarm is that it it <br>
56:34 <br>
does add <br>
56:35 <br>
an extra layer of security and and and <br>
56:38 <br>
and decentralization even in the face of <br>
56:41 <br>
having having your node running <br>
56:44 <br>
because at least it obfuscates all the <br>
56:46 <br>
content specific <br>
56:47 <br>
and an application specific uh <br>
56:52 <br>
logic so so that if if a small node has <br>
56:56 <br>
a basic infrastructure ingredient is <br>
56:58 <br>
allowed to <br>
57:00 <br>
run in general on on amazon <br>
57:03 <br>
then on top of that there's no there's <br>
57:05 <br>
no proper <br>
57:06 <br>
potential mechanism to censor or ban or <br>
57:10 <br>
drop <br>
57:21 <br>
yeah i think i think that's i think <br>
57:23 <br>
that's uh very very good that you say <br>
57:25 <br>
that <br>
57:25 <br>
daniel to make that distinction uh i <br>
57:28 <br>
specifically wanted to talk not about <br>
57:30 <br>
what we have today <br>
57:31 <br>
but with what the vision is and where we <br>
57:33 <br>
want to go and why then maybe aws is not <br>
57:36 <br>
the best idea <br>
57:37 <br>
i hope i hope i'm not opening a can of <br>
57:39 <br>
worms here yeah but in <br>
57:41 <br>
this anything it's like complete sense <br>
57:42 <br>
it's this food like if amazon can can <br>
57:44 <br>
shut down <br>
57:45 <br>
the particular service of of an <br>
57:48 <br>
application or a development team <br>
57:50 <br>
but if if they want to shut down the <br>
57:53 <br>
particular application or development <br>
57:55 <br>
of an app on swarm then what they can do <br>
57:58 <br>
is <br>
57:58 <br>
to shut down all the small nodes <br>
58:04 <br>
yeah now quite clear i think i hope this <br>
58:07 <br>
was clear for <br>
58:08 <br>
for our viewers i would i always send <br>
58:10 <br>
people to <br>
58:11 <br>
that want to understand this more i <br>
58:13 <br>
always tell them like at least <br>
58:14 <br>
read the introduction in the book of <br>
58:16 <br>
swarm because you know it explains <br>
58:18 <br>
so well um what the history is of what <br>
58:22 <br>
we are trying to achieve and and why we <br>
58:24 <br>
are doing it <br>
58:25 <br>
um and i think that's a very important <br>
58:27 <br>
aspect of swarm <br>
58:28 <br>
that people need to understand if they <br>
58:30 <br>
invest time and energy and bus stops and <br>
58:33 <br>
running notes <br>
58:34 <br>
um i'll take this as the last question <br>
58:37 <br>
if that's okay with you guys um it seems <br>
58:41 <br>
that downloads don't work correctly even <br>
58:44 <br>
though the upload was marked as <br>
58:46 <br>
completed <br>
58:47 <br>
what gives and isn't this a critical <br>
58:49 <br>
issue for mainnet <br>
58:54 <br>
um yeah i mean of course they're asking <br>
58:57 <br>
like <br>
58:57 <br>
does it work yeah i mean this is this is <br>
59:00 <br>
of course a very critical issue and <br>
59:02 <br>
um it ties into like <br>
59:05 <br>
what we've been working on basically for <br>
59:07 <br>
the last year <br>
59:09 <br>
um so the swarm is um <br>
59:13 <br>
a rather complex piece of software that <br>
59:15 <br>
has been operating <br>
59:16 <br>
on hardware that we don't know and <br>
59:18 <br>
really like operating all <br>
59:20 <br>
across the globe um <br>
59:21 <br>
[Music] <br>
59:23 <br>
having all the bits and pieces fall into <br>
59:25 <br>
place <br>
59:26 <br>
is is rather complex and we have been <br>
59:28 <br>
observing indeed as well <br>
59:30 <br>
if you upload something to the network <br>
59:32 <br>
then somehow you cannot get it back <br>
59:34 <br>
um i'm very i'm rather confident <br>
59:39 <br>
about the changes that we are going to <br>
59:40 <br>
release today <br>
59:42 <br>
and really curious to see <br>
59:45 <br>
how the network will operate and perform <br>
59:47 <br>
my own <br>
59:48 <br>
observations have been that if i upload <br>
59:51 <br>
a small size <br>
59:52 <br>
i can download it again but if i upload <br>
59:56 <br>
a bigger <br>
59:56 <br>
size you know file usually i couldn't <br>
59:59 <br>
download <br>
60:00 <br>
it and this is because of you know just <br>
60:03 <br>
statistics right so if i upload <br>
60:05 <br>
1000 junks if one of those chunks is <br>
60:08 <br>
missing <br>
60:08 <br>
the file will be corrupt so really if <br>
60:11 <br>
you upload bigger <br>
60:12 <br>
sizes files all the chunks really need <br>
60:15 <br>
to become in their place <br>
60:16 <br>
and for this reason we advise the limits <br>
60:20 <br>
on the upload size so we advise to do <br>
60:23 <br>
swarm <br>
60:24 <br>
in beginning with smaller size files <br>
60:27 <br>
just because the <br>
60:29 <br>
the probability of like being able to <br>
60:31 <br>
retrieve all those jumps into those <br>
60:32 <br>
files <br>
60:33 <br>
is is higher and we work towards <br>
60:35 <br>
increasing these limits <br>
60:37 <br>
over the time okay that's that's a very <br>
60:40 <br>
clear <br>
60:41 <br>
um and comprehensive answer it reminds <br>
60:43 <br>
me of <br>
60:44 <br>
when i started to to do the internet <br>
60:47 <br>
thing like at first it was even like <br>
60:49 <br>
slow to download an image and you had to <br>
60:51 <br>
wait like before <br>
60:52 <br>
you know before it was was downloaded um <br>
60:56 <br>
to me a lot of what we are doing reminds <br>
60:58 <br>
me of <br>
60:59 <br>
early days of the internet etc lsc we <br>
61:02 <br>
are losing victor victor thank you so <br>
61:03 <br>
much for this ama <br>
61:07 <br>
okay and uh guys i want every one of you <br>
61:11 <br>
so found out <br>
61:12 <br>
there were not many questions about <br>
61:13 <br>
infrastructure i'd see this as a good <br>
61:16 <br>
sign <br>
61:16 <br>
um and uh but thank you so much that you <br>
61:19 <br>
wanted to be here today to to answer <br>
61:21 <br>
these questions and just you know just <br>
61:23 <br>
to show your face i think it's important <br>
61:25 <br>
for our users and for node operators for <br>
61:27 <br>
investors that they have the opportunity <br>
61:29 <br>
to also meet <br>
61:30 <br>
the team and to see that you know that <br>
61:32 <br>
you guys are like <br>
61:34 <br>
you know what you're talking about <br>
61:35 <br>
basically um so <br>
61:37 <br>
i want to thank you all rinke um gregor <br>
61:40 <br>
danny as always um i'm gonna see you <br>
61:43 <br>
guys really really soon in budapest i'm <br>
61:45 <br>
looking forward to that <br>
61:46 <br>
with that i would like to round up the <br>
61:48 <br>
ama please uh <br>
61:49 <br>
everyone viewing if your quest question <br>
61:52 <br>
was not answered just go to our discord <br>
61:53 <br>
channel <br>
61:54 <br>
go to node operators to be support we <br>
61:56 <br>
have dedicated channels <br>
61:58 <br>
um where people will uh in a timely <br>
62:00 <br>
fashion answer <br>
62:01 <br>
your questions so for now i say goodbye <br>
62:05 <br>
from antwerp and i'll see you soon in <br>
62:06 <br>
budapest <br>
62:07 <br>
[Music] <br>
62:18 <br>
bye <br>
62:34 <br>
you <br>