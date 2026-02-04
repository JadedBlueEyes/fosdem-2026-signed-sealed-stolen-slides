---
# try also 'default' to start simple
theme: apple-basic
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
# background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: Signed, Sealed, Stolen
info: |
  What happens when your server starts signing messages you didn't send?

  Recently, the [Continuwuity project](https://continuwuity.org) (a Rust-based [Matrix](https://matrix.org) homeserver) fell victim to a targeted, active exploitation campaign. Attackers leveraged two critical vulnerabilities (CVSS [9.9](https://github.com/continuwuity/continuwuity/security/advisories/GHSA-22fw-4jq7-g8r8) and [9.3](https://github.com/continuwuity/continuwuity/security/advisories/GHSA-m5p2-vccg-8c9v)) affecting the entire ecosystem of [Conduit](https://conduit.rs/)-derived servers. By exploiting flaws in the way that servers join and leave chat rooms, attackers forced the server to cryptographically sign unexpected events, with disasterous results. This allowed them to forge "leaves" to decimate public rooms, forge ACL rules to brick them, and temporarily take over an account to exfiltrate over 5,000 messages from the maintainers' private internal chat.

  In this talk, Nex and Jade will take you inside the war room during the incident. We'll walk through the attack chain, explaining how attackers tricked the server, and how we figured out what happened. We'll also have a brief look at how we hardened our project against similar exploitation in the future.
# apply UnoCSS classes to the current slide
# class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: true
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# duration of the presentation
duration: 15min

download: true

addons:
  - slidev-addon-sync
layout: center
dragPos:
  logo: 586,19,159,156,-16
canvasWidth: 800
---

# Signed, Sealed, Stolen
## How We Patched Critical Vulnerabilities Under Fire
### Jade, Nexus


<img v-drag="'logo'" src="./img/logo.svg">

---

<div flex>
    <div flex-1 />
    <div
      v-click="1" flex flex-col items-center transition duration-500 ease-in-out
      :class="$clicks < 1 ? 'translate-y-20 opacity-0' : 'translate-y-0 opacity-100'"
    >
        <img src="/img/jade.svg" w-50 h-50 rounded-full object-cover mb-5>
        <span text-3xl>Jade Ellis</span>
        <div items-center>
            <div text-sm flex items-center justify-center gap-2 mt-4>
                <a href="https://jade.ellis.link/">jade.ellis.link</a>
            </div>
            <div text-sm flex items-center justify-center gap-2 mt-4>
                <div i-ri:github-fill /><a href="https://github.com/JadedBlueEyes" font-mono >JadedBlueEyes</a>
            </div>
        </div>
    </div>
    <div flex-1 />
    <div
        v-click="2" flex flex-col items-center transition duration-500 ease-in-out
        :class="$clicks < 2 ? 'translate-y-20 opacity-0' : 'translate-y-0 opacity-100'"
    >
        <img src="/img/nex.webp" w-50 h-50 rounded-full object-cover mb-5>
        <span text-3xl >Nexus</span>
        <div items-center>
            <div text-sm flex items-center justify-center gap-2 mt-4>
                <a href="https://timedout.uk/">timedout.uk</a>
            </div>
            <div text-sm flex items-center justify-center gap-2 mt-4>
                <div i-ri:github-fill /><a href="https://github.com/nexy7574/" font-mono >nexy7574</a>
            </div>
        </div>
    </div>
    <div flex-1 />
</div>

---
layout: fact
---

## We are not CyberSecurity professionals

---

## Continuwuity

<v-clicks>

- Continuwuity is a matrix Chat server
- continuwuity.org is where we keep 'Official' secondary accounts
- Continuwuity is a part of an ecosystem of Conduit-based servers, forked from the same codebase
    - Conduit
    - Grapevine
    - Tuwunel 

</v-clicks>

---
layout: section
---

## Setting the scene

<!-- 
it's 4:00am
Downloading books
See a message I didn't send
-->

---
layout: image-left
image: /uh oh.png
backgroundSize: contain
---

## This is very bad

- Jade did not send this event
- It contains a command which prints out every session token for the account
- There are a bunch of weird things going on with the event

---
layout: fact
---

<span text-xl bold>23 seconds</span>

---

## What's the first thing you do when an account is compromised?


<v-clicks>

- Alert your team
- Panic
- Disable the account's admin access
- Suspend the account
- Reset the password 
- Disable all login tokens
- Pull out the plugs on other possible targets

</v-clicks>

---
layout: center
---

<v-clicks>

## OK, so we have to be safe now.

## But how did the attacker do that?

</v-clicks>

---


## Time to investigate

- Identifying the attacking server is easy
- But the event seems to check out, there's nothing suspicious from a remote server
- I have a hunch

---

## The incriminating event

```json
{
  "auth_events": [
    "$R-PRPFumZ6zfsmW4Ek0YkvpeCdtAWRVlMWzol5aNgQA",
    "$HMbtQIwqzbMKJCHM4LxhYY31byrU3mj2SKphMJhEnSQ",
    "$CeVk90j0crDU47l098iDOIRDME4AiNwV0bkfw7F3jyo"
  ],
  "content": {
    "body": "\\!admin query raw raw-iter userdeviceid_token @jade",
    "m.relates_to": {
      "event_id": "$R-PRPFumZ6zfsmW4Ek0YkvpeCdtAWRVlMWzol5aNgQA",
      "rel_type": "m.thread"
    },
    "msgtype": "m.notice"
  },
  "depth": 70,
  "event_id": "$f8r_Zq-hVfKSgLMkeNB7IXTtVbO85DFELqpT5R6N8kU",
  "hashes": {
    "sha256": "B+1hum2uRiJ/hnX+sHTlGP+yWPNHyNyPvHjzHrJo9Qo"
  },
  ...
```

---

```json
...
  "origin": "continuwuity.org",
  "origin_server_ts": 1767067175446,
  "prev_events": [
    "$Cabj9hUltCn2vDgAonj7tLzwR4eSE6e58N2bjh7nObE"
  ],
  "room_id": "!3og6vwG8x7aZeCXJvC:ellis.link",
  "sender": "@jade:continuwuity.org",
  "signatures": {
    "continuwuity.org": {
      "ed25519:PwHlNsFu": "Oyd4XIgJbrQ/HMevRVHyWjoZXndOQCZFXiKRjXdNbtjyZKugHyGHNsRCRjQ4aH3EiPJrn9NocNBhJ6A2JM7lDA"
    }
  },
  "type": "m.room.message"
}
```


---
layout: quote
zoom: 1.75
---

> that only happens for events not sent via /send <span text-xs>[04:30]</span>

The event must be forged over federation.

---

## In the meantime...

The attacker is causing as much damage as possible, now they're locked out of Jade's account

<img src="./img/Screenshot 2026-02-01 at 11.49.33.png">
    
One by one, our public chatrooms are made unusable


---
layout: center
---

The time is now 5:00. It has been one hour.

---

## Jade's hot tip: Don't talk to hackers


---

## The next insight

<v-clicks>

- We have access to logs from other servers
- Based on failed attempts to access private rooms, the attacker requires knowlege of 'auth events' (ginger)
- These are previous messages in a room that control permissions
- The attacker must have an account already in the room
- The attacker must handcraft the events

</v-clicks>

---
layout: center
---

`[05:24] One maintainer down`

---
layout: center
---

`[06:10] Two maintainers down`


---
layout: center
---

`[06:30] One maintainer up`


---

## Free data

<v-clicks>

\[07:13] The attacker starts probing a server we have debug logging and instrumentation on.

We now know the vulnerability.


`[07:57] One maintainer down.`

</v-clicks>

---

## Before we get to the vulnerability: how do you leave a room?


<v-clicks>

- To leave a room, your server must construct a leave event
- Making this requires full knowledge of the current state of the room
- But the leaving server doesn't always have it
- The leaving server selects a server it knows is in the room, and asks it to make the leave event for it
- `GET /_matrix/federation/v1/make_leave/{roomId}/{userId}` returns the leave event.
- The leaving server then signs the event and returns it to the same server

</v-clicks>

---

```rust
    for remote_server in servers {
		let make_leave_response = services
			.sending
			.send_federation_request(
				remote_server.as_ref(),
				federation::membership::prepare_leave_event::v1::Request {
					room_id: room_id.to_owned(),
					user_id: user_id.to_owned(),
				},
			)
			.await;
```

---


```rust

	let mut leave_event_stub = serde_json::from_str::<CanonicalJsonObject>(
		make_leave_response.event.get(),
	)
	.map_err(|e| {
		err!(BadServerResponse(warn!(
			"Invalid make_leave event json received from {remote_server} for {room_id}: {e:?}"
		)))
	})?;

	// TODO: Is origin needed?
	leave_event_stub.insert(
		"origin".to_owned(),
		CanonicalJsonValue::String(services.globals.server_name().as_str().to_owned()),
	);
	leave_event_stub.insert(
		"origin_server_ts".to_owned(),
		CanonicalJsonValue::Integer(
			utils::millis_since_unix_epoch()
				.try_into()
				.expect("Timestamp is valid js_int value"),
		),
	);
```


---


```rust
	// In order to create a compatible ref hash (EventID) the `hashes` field needs
	// to be present
	services
		.server_keys
		.hash_and_sign_event(&mut leave_event_stub, &room_version_id)?;

```

```rust
	services
		.sending
		.send_federation_request(
			&remote_server,
			federation::membership::create_leave_event::v2::Request {
				room_id: room_id.to_owned(),
				event_id: event_id.clone(),
				pdu: services
					.sending
					.convert_to_outgoing_federation_event(leave_event.clone())
					.await,
			},
		)
		.await?;
```

---

<span text-xl>Yeah.</span>

---

<img src="./img/Screenshot 2026-02-01 at 13-03-44 validate membership events returned by remote servers · 12aecf8091 - continuwuation_continuwuity - Ellis Git.png">

---

```rust
    validate_remote_member_event_stub(
			&MembershipState::Leave,
			user_id,
			room_id,
			&leave_event_stub,
		)?;
```

```rust

	/// Validates that an event returned from a remote server by `/make_*`
	/// actually is a membership event with the expected fields.
	///
	/// Without checking this, the remote server could use the remote membership
	/// mechanism to trick our server into signing arbitrary malicious events.
	pub(crate) fn validate_remote_member_event_stub(
		membership: &MembershipState,
		user_id: &UserId,
		room_id: &RoomId,
		event_stub: &CanonicalJsonObject,
	) -> Result<()> {
```

---

```rust
    let Some(event_type) = event_stub.get("type") else {
			return Err!(BadServerResponse(
				"Remote server returned member event with missing type field"
			));
		};
		if event_type != &RoomMemberEventContent::TYPE {
			return Err!(BadServerResponse(
				"Remote server returned member event with invalid event type"
			));
		}
```

---

```rust
    let Some(sender) = event_stub.get("sender") else {
			return Err!(BadServerResponse(
				"Remote server returned member event with missing sender field"
			));
		};
		if sender != &user_id.as_str() {
			return Err!(BadServerResponse(
				"Remote server returned member event with incorrect sender"
			));
		}
```

---

```rust
		let Some(state_key) = event_stub.get("state_key") else {
			return Err!(BadServerResponse(
				"Remote server returned member event with missing state_key field"
			));
		};
		if state_key != &user_id.as_str() {
			return Err!(BadServerResponse(
				"Remote server returned member event with incorrect state_key"
			));
		}
```

---

```rust
		let Some(event_room_id) = event_stub.get("room_id") else {
			return Err!(BadServerResponse(
				"Remote server returned member event with missing room_id field"
			));
		};
		if event_room_id != &room_id.as_str() {
			return Err!(BadServerResponse(
				"Remote server returned member event with incorrect room_id"
			));
		}
```

---

```rust
		let Some(content) = event_stub
			.get("content")
			.and_then(|content| content.as_object())
		else {
			return Err!(BadServerResponse(
				"Remote server returned member event with missing content field"
			));
		};
		let Some(event_membership) = content.get("membership") else {
			return Err!(BadServerResponse(
				"Remote server returned member event with missing membership field"
			));
		};
		if event_membership != &membership.as_str() {
			return Err!(BadServerResponse(
				"Remote server returned member event with incorrect membership"
			));
		}
```

---

## OK, so how did they trigger the exploit?


<v-clicks>

`@bot:continuwuity.org`

```python
@event.on(EventType.ROOM_MEMBER)
async def handle_invite(self, evt: StateEvent) -> None:
    # i'm allergic to and statements
    if evt.state_key == self.client.mxid:
        if evt.content.membership == Membership.INVITE:
            if self.is_user_trustworthy(evt.sender):
                await self.client.join_room(evt.room_id)
            else:
                await self.client.leave_room(evt.room_id)
```

</v-clicks>

---

## Distributing the patch

- 3 other projects are vulnerable in the same way
- Both the spec and another project are vulnerable to a lesser degree
- Some projects we have channels with, some we don't

---
layout: center
---

## Release complete

Time to completion: 13h45m

---
layout: section
---

## Let's travel back in time

---

03:12:02: The attacker joins a Matrix room dedicated to a side project of mine.

03:59:35: The attacker sends a forged message as my continuwuity.org account with the content \\!admin query raw raw-iter userdeviceid_token @jade. This invocation of the “escaped admin commands” feature—enabled by default—causes that account to respond with the contents of the table: namely, all of the login tokens for @jade:continuwuity.org. This is the first visible evidence of an attack.

04:00:44: The attacker redacts the result with curl

04:01:32: The attacker uses the stolen token to log in with Element Web

---


04:03:00: The attacker begins exporting messages from the moderation room

04:03:39: I leave the continuwuity.org admin room.
04:03:41: Nex suspends the account.
04:03:53: Nex force-resets the password for the account. This doesn’t log out any devices.

04:05:40: The attacker stops exporting messages from the moderation room  
04:06:24-44: Attacker tries to view direct messages between @jade:continuwuity.org and other accounts, is stopped by e2ee  
04:06:56: Attacker opens team room and begins exporting messages  
04:07:47: Attacker is stopped mid-export by session deactivation  

---

## How we're securing continuwuity against similar attacks

- Account locking
- Admin command to forcefully log out all of a user's existing sessions
- Configuration-defined admins
- Disabling account logins
- Hardening admin command escapes
- Restricting certain admin commands
- Project Drawbridge: over 100 commits hardening federation APIs 


---


---
