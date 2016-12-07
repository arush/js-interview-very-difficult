The frontend is still using tartan for Plaid

[11:14]  
however, it’s configured correctly

arush [11:31 AM]  
@anze @spacenick this line is resolving to `tartan` because envify is taking TRY_PLAID_TEST_MODE=true as the string "true" instead of boolean true
`process.env.TRY_PLAID_TEST_MODE === 'false' ? 'prod' : 'tartan',`

anze [11:32 AM]  
but it’s comparing a string

arush [11:32 AM]  
oh yeah

anze [11:32 AM]  
there is no booleans in env variables

[11:32]  
so it should be fine

[11:32]  
how do we inspect the actual value of it in the browser?

arush [11:32 AM]  
its definitely resolving to tartan in the browser, i just tested it

anze [11:32 AM]  
yeah I did too

arush [11:35 AM]  
@anze i'm inspecting it in the browser using breakpoints

anze [11:35 AM]  
ok

[11:35]  
can you get that process.env value out?

arush [11:35 AM]  
yeah

[11:36]  
looking for it now

[11:36]  
i have to look through the uglified js so tis taking a while

[11:40]  
oh of course

[11:41]  
no wait

anze [11:41 AM]  
{
                               "name": "TRY_PLAID_TEST_MODE",
                               "value": "false"
                           },

arush [11:41 AM]  
so far found this

anze [11:41 AM]  
it’s set to false

arush [11:41 AM]  
`env:  false ? 'prod' : 'tartan',`

anze [11:41 AM]  
hm

arush [11:41 AM]  
this is in the uglified javascript

[11:42]  
which is produced by envify during our build step. im going to keep digging to find the env key value store in the browser

anze [11:43 AM]  
we can just hardcode prod for now

arush [11:43 AM]  
ok there is no such store

anze [11:43 AM]  
as a quick workaround

arush [11:43 AM]  
because this line is evaluated at build time

anze [11:44 AM]  
it can’t be at build time

[11:44]  
because the value isn’t set until it’s run

[11:45]  
we actually pass all config in at runtime

arush [11:46 AM]  
ok then i'll find it

anze [11:46 AM]  
https://github.com/trycom/frontend/blob/master/try.com/docker-resources/run#L40

[11:46]  
we do a simple search and replace here

arush [11:47 AM]  
another clue i found

[11:48]  
is the TRY_PLAID_PUBLIC_KEY the same for prod and test?

[11:48]  
cos that is resolving to: `[REDACTED]`

[11:51]  
here is what our js env is feeding from

```{
                            TRY_API_URL: "//api.try.com",
                            TRY_STACK_NAME: "prod",
                            TRY_SUPPORT_START_UTC: "3:00pm",
                            TRY_SUPPORT_END_UTC: "1:00am",
                            TRY_SUPPORT_WEEKEND_START_UTC: "5:00pm",
                            TRY_SUPPORT_WEEKEND_END_UTC: "1:00am",
                            TRY_PLAID_PUBLIC_KEY: "[REDACTED]",
                            TRY_PLAID_TEST_MODE: "false",
                            NODE_ENV: "production",
                            BLUEBIRD_WARNINGS: 0,
                            BLUEBIRD_DEBUG: 0
                        }
```

arush [12:10 PM]  
@anze that find and replace is looking for $TRY_PLAID_TEST_MODE which I believe is different to TRY_PLAID_TEST_MODE (without the $) ...? in any case, pretty sure this line is evaluated at an earlier point by envify. the only source code the browser can see is this:

try-[hash].js
```r.plaidInjectionPromise.then(function() {
  return (0, v.openPlaidModal)({
    env: "tartan",
    clientName: r.props.currentUser.firstName + " " + r.props.currentUser.lastName,
    key: "[REDACTED]"
  })
})
```

webpack://try-[hash].js (source mapped from above)
```_this.plaidInjectionPromise.then(function () {
  return (0, _plaid.openPlaidModal)({
    env:  false ? 'prod' : 'tartan',
    clientName: _this.props.currentUser.firstName + ' ' + _this.props.currentUser.lastName,
    key: ("$TRY_PLAID_PUBLIC_KEY")
  });
})
```

[12:12]  
you can see the $ var has not been replaced yet, but process.env.TRY_PLAID_TEST_MODE has already been evaluated

anze [12:12 PM]  
hm that seems odd

[12:12]  
our generated js files should retain $VARIABLE_NAME (edited)

arush [12:13 PM]  
yeah it is odd

anze [12:13 PM]  
we did this ages ago, because we don’t want to bake config in at compile time

[12:14]  
did you looks at those sources in the browser?

[12:14]  
if so, it’s correct that the values have been replaced

arush [12:14 PM]  
yes this is from the browser

anze [12:15 PM]  
ok

[12:15]  
then the value passed forward ok

[12:15]  
it’s “false"

arush [12:15 PM]  
yea

anze [12:16 PM]  
```TRY_FACEBOOK_APP_ID: "$TRY_FACEBOOK_APP_ID",
                            TRY_SEGMENT_API_KEY: "$TRY_SEGMENT_API_KEY",```
and `ANALYTICS_EVENT_VERSION: "$ANALYTICS_EVENT_VERSION"`

[12:16]  
so I guess these are no longer used

anze [12:28 PM]  
is frontend master clean and safe to deploy?

arush [12:29 PM]  
i haven't touched it

[12:29]  
so should be as nick left it

anze [12:30 PM]  
you commited 10,460 additions and 1,648 deletions 1h ago

[12:31]  
and I don’t see that in develop

arush [12:31 PM]  
yeah i wouldn't deploy that

anze [12:31 PM]  
ok

arush [12:31 PM]  
probably need to undo that commit because i probably merged branches locally

[12:32]  
and nick has been managing these branches separately

[12:32]  
is prod try.com on master or develop?

anze [12:32 PM]  
master

arush [12:32 PM]  
yeah def need to unwind that

anze [12:33 PM]  
do you guys just rebase or revert?

arush [12:33 PM]  
revert

anze [12:33 PM]  
safer :slightly_smiling_face:

[12:34]  
I’ll hardcode prod env for now

[12:34]  
Nick can solve the mystery tomorrow

arush [12:34 PM]  
you can just hardcode that one line to "prod" instead of tartan

anze [12:35 PM]  
yeah

[12:36]  
it seems reverting merges is complicated https://github.com/git/git/blob/master/Documentation/howto/revert-a-faulty-merge.txt
 GitHub
git/git
git - Git Source Code Mirror - This is a publish-only repository and all pull requests are ignored. Please follow Documentation/SubmittingPatches procedure for any of your improvements. 
 
 

arush [12:37 PM]  
its prob not necessary, it was prob a safe merge, but

[12:37]  
just don't wanna do it without nick confirming

carlos [12:37 PM]  
you need to branch out, and reset to a safe commit.

anze [12:38 PM]  
but a reset would require a forced push

[12:38]  
wouldn’t it?

carlos [12:38 PM]  
yep

[12:38]  
That's why I said branch out

[12:38]  
just branch out the current work

[12:38]  
(merged work)

[12:38]  
and reset to a safe commit.

anze [12:38 PM]  
that makes sense

carlos [12:38 PM]  
Review the branch out point

[12:38]  
later

[12:38]  
but don't lose the commit reference.

[12:39]  
A branch is just a commit reference.

[12:39]  
You guys need to define what is a point of a safe commit to reset to.

anze [12:40 PM]  
HEAD~2 (edited)

anze [1:27 PM]  
github is weird today, it’s not showing my push

arush [2:04 PM]  
@anze can you pls deploy master to prod, i hardcoded the env and did some logging

anze [2:05 PM]  
will do

arush [2:05 PM]  
tx

[2:07]  
@anze this should be the one 13a66d51

anze [2:08 PM]  
ok

arush [2:18 PM]  
build is successful

anze [2:19 PM]  
yep, I saw

[2:19]  
deployed

ankush [2:26 PM]  
```[SAME AS ABOVE]
```
(edited)

arush [2:28 PM]  
that is the result of console.log(process.env) in production

[2:29]  
which i guess has come from our webpackConfigGenerator

ankush [2:37 PM]  
set up a reminder “Daily Standup https://zoom.us/j/924727197” in this channel at 9am every weekday, Pacific Standard Time.

ankush [2:38 PM]  
set up a reminder “Complete your YTB for Standup: https://paper.dropbox.com/doc/Eng.-Standups-0Dw3NX0yY5SoGfFv4sAlD” in this channel at 8:45am every weekday, Pacific Standard Time.

arush [4:43 PM]  
@anze any reason why incomes, income streams and risks would not have come through on these people that linked Plaid?

anze [4:43 PM]  
yep

[4:43]  
already a card for it

[4:48]  
I assigned it to @carlos: https://www.pivotaltracker.com/story/show/135659089

arush [4:52 PM]  
cool

spacenick [11:42 PM]  
there's nothing wrong with the webpack config/environment as you probably noticed

arush [11:48 PM]  
@spacenick yeah, we can't figure it out, would love to know what you find

spacenick [11:49 PM]  
well you already found out the issue, the plaid enviroment is 'production' not prod

[11:49]  
otherwise the logic that was there was fine

arush [11:49 PM]  
no it was rendering tartan

[11:49]  
env: 'tartan'

[11:50]  
https://trycom.slack.com/archives/engineering/p1481055058000063
 arush
@anze that find and replace is looking for $TRY_PLAID_TEST_MODE which I believe is different to TRY_PLAID_TEST_MODE (without the $) ...? in any case, pretty sure this line is evaluated at an earlier point by envify. the only source code the browser can see is this:

try-[hash].js
```r.plaidInjectionPromise.then(function() {
  return (0, v.openPlaidModal)({
    env: "tartan",
    clientName: r.props.currentUser.firstName + " " + r.props.currentUser.lastName,
    key: "[REDACTED]"
  })
})
```

webpack://try-[hash].js (source mapped from above)
```_this.plaidInjectionPromise.then(function () {
  return (0, _plaid.openPlaidModal)({
    env:  false ? 'prod' : 'tartan',
    clientName: _this.props.currentUser.firstName + ' ' + _this.props.currentUser.lastName,
    key: ("$TRY_PLAID_PUBLIC_KEY")
  });
})
```
Posted in #engineeringYesterday at 12:10 PM 

[11:51]  
breakpoint investigation also revealed that the variable was tartan not prod

[11:54]  
anyway not a huge priority but very curious to find out what is going on


----- Today December 7th, 2016 -----
spacenick [12:02 AM]  
this was uglifyJS trying to optimize code at compile time

[12:03]  
it will evaluate comparison to remove code that is under 'always false' blocks

[12:03]  
the gist is not to use direct comparison on process.env variable but to use functions

arush [12:03 AM]  
interesting

[12:03]  
that makes total sense

[12:03]  
how did you find that out?

spacenick [12:03 AM]  
eh that's the only thing that could be the cause really

[12:03]  
the value of the env variable was correct

arush [12:04 AM]  
makes total sense

spacenick [12:04 AM]  
the code you pasted from the webpack:// file gives the best hint here

arush [12:04 AM]  
yup

spacenick [12:04 AM]  
bceause $TRY_PLAID_PUBLIC_KEY wasn't evaluated but the env was already set, so it had to be at the webpack level

[12:04]  
should be good now

[12:05]  
I'm actually just surprised I didn't run into this earlier, but I don't think we store that many true/false variables as env (mostly keys)

arush [12:06 AM]  
:thumbsup: cool so our rule when writing js is to always use functions to get env vars?

spacenick [12:07 AM]  
env vars are fine in themselves, but evaluations should be wrapped in a function to be evaluated at runtime

[12:07]  
hmm

[12:08]  
even that doesn't make complete sense

[12:08]  
because we're already wrapped into so many functions

[12:10]  
i think this is a really peculiar case of setting an object attribute from inline evaluating

[12:12]  
ok yeah this fixed it https://github.com/trycom/frontend/commit/988d36385515e1d1a98220e8a59a24cb3d870023

[12:12]  
your investigation helped to save lots of time so thanks for that

arush [12:16 AM]  
nice. that's pretty aggressive from uglify

spacenick [12:17 AM]  
yep

arush [12:17 AM]  
ok so setting object attribute from inline env var eval is not good

spacenick [12:18 AM]  
well, we're also using a quite specific method (setting those static '$VAR' variables and replacing them at runtime) which is definitely not common in that realm

[12:19]  
yeah and I think that if you write
```if (process.env.MY_VAR === 'something')
```

this is also gonna be wiped away by uglifyJS

[12:19]  
because at that point process.env.MY_VAR === '$MY_VAR'

[12:19]  
so safer to wrap into if (myComparison()) so that it doesn't get uglified (uglifyjs understands it in that case)

[12:20]  
I guess this is one gotcha of our stack here, I'll take note to write something about it

arush [12:20 AM]  
ok cool

[12:20]  
tbh your comments and readme in the webpack config is a good place for this

[12:21]  
that is some really :thumbsup: developer documentation

spacenick [12:21 AM]  
yeah good point I'll add a note there

arush [12:21 AM]  
this would make an aweseome interview question if we can describe the scenario without giving away the answer :wink:

new messages
spacenick [12:22 AM]  
hehe yeah this is def a tricky one
