# ASCII Alert Overlay

A transparent OBS overlay that draws a message in an ASCII-art font on a 320 by 96 character screen.

The message is centered. A block cursor sweeps across each row and the characters appear as it passes, like a terminal receiving the screen. The cursor then blinks at the end of the message. After 10 seconds, it drops to the bottom line and carriage returns scroll the screen up until the message has left the top.

Font, message, and color are sent from Streamer.bot, so the same overlay can show follows, subscriptions, raids, or anything else you want to trigger.

## Install

In OBS, add a Browser source.

Use a **local file**, not the GitHub Pages link. The overlay has to open a websocket to Streamer.bot on this PC, and an `https://` page will block that.

- URL: `file:///D:/repos/streamer-tools/ascii/ascii.html`
- Width: 1920
- Height: 1080

Leave the background transparent. Leave the source **visible**. It draws nothing until an alert arrives. Do not enable **Shutdown source when not visible**, or it will miss the Streamer.bot trigger while it is hidden.

To try it in a normal browser, open `ascii.html?preview=1`. Click the page, or press Space, to cycle sample alerts. Esc dismisses the one on screen.

## Streamer.bot

1. Open **Servers / Clients → WebSocket Server** and start it. The default is `127.0.0.1` port `8080`. Leave authentication off on a trusted PC, or add `&password=` to the browser source URL.
2. Create an action for the trigger you want (Follow, Subscription, Raid, a command, a Stream Deck button, and so on).
3. Add a sub-action: **Core → C# → Execute C# Code**.
4. Paste one of the snippets below. Do not wrap it in a class.

The overlay only reacts when `event` is `ascii`, so it can share the websocket server with the system error overlay.

Alerts play one at a time. If another arrives while one is on screen, it waits. Up to 6 can wait; a 7th replaces the oldest one still waiting.

### Follow

Add this to the Twitch **Follow** trigger. `user` is the display name.

```csharp
string name = "someone";
if (args.ContainsKey("user") && args["user"] != null && args["user"].ToString() != "")
    name = args["user"].ToString();
else if (args.ContainsKey("userName") && args["userName"] != null)
    name = args["userName"].ToString();
name = name.Replace("\\", "\\\\").Replace("\"", "\\\"").Replace("\r", "").Replace("\n", " ");
CPH.WebsocketBroadcastJson("{\"event\":\"ascii\",\"font\":\"block\",\"color\":\"33ff66\",\"message\":\"" + name + "\\nFOLLOWED\"}");
```

### Subscription

Add this to **Subscription** and **Resubscription**. A first sub says `SUBSCRIBED`. A resub says the cumulative months, for example `5 MONTHS`.

```csharp
string name = "someone";
if (args.ContainsKey("user") && args["user"] != null && args["user"].ToString() != "")
    name = args["user"].ToString();
else if (args.ContainsKey("userName") && args["userName"] != null)
    name = args["userName"].ToString();
string kind = "SUBSCRIBED";
if (args.ContainsKey("cumulative") && args["cumulative"] != null)
{
    int months;
    if (int.TryParse(args["cumulative"].ToString(), out months) && months > 1)
        kind = months + " MONTHS";
}
name = name.Replace("\\", "\\\\").Replace("\"", "\\\"").Replace("\r", "").Replace("\n", " ");
kind = kind.Replace("\\", "\\\\").Replace("\"", "\\\"");
CPH.WebsocketBroadcastJson("{\"event\":\"ascii\",\"font\":\"block\",\"color\":\"ffcc33\",\"message\":\"" + name + "\\n" + kind + "\"}");
```

### Raid

Add this to the Twitch **Raid** trigger (the one that fires when you are raided). `viewers` is the raid size.

```csharp
string name = "someone";
if (args.ContainsKey("user") && args["user"] != null && args["user"].ToString() != "")
    name = args["user"].ToString();
else if (args.ContainsKey("userName") && args["userName"] != null)
    name = args["userName"].ToString();
string viewers = "";
if (args.ContainsKey("viewers") && args["viewers"] != null)
    viewers = args["viewers"].ToString();
string line1 = viewers == "" ? "RAID" : "RAID " + viewers;
name = name.Replace("\\", "\\\\").Replace("\"", "\\\"").Replace("\r", "").Replace("\n", " ");
line1 = line1.Replace("\\", "\\\\").Replace("\"", "\\\"");
CPH.WebsocketBroadcastJson("{\"event\":\"ascii\",\"font\":\"graffiti\",\"color\":\"ff3355\",\"message\":\"" + line1 + "\\n" + name + "\"}");
```

`graffiti` is the banner font (the same face as Graffiti on [TAAG](https://patorjk.com/software/taag/)). A name that cannot fit on one line steps down to a narrower font so the word is not split. `grafiti` is accepted as a spelling of `graffiti`.

### Test from chat

Create a command such as `!ascii` and paste this. The rest of the message is what gets drawn. `font:` and `color:` are optional. `|` starts a new line.

`!ascii font:graffiti color:gold RAID|KEVIN`

`!ascii dismiss` clears the overlay.

```csharp
string text = "";
if (args.ContainsKey("rawInput") && args["rawInput"] != null)
    text = args["rawInput"].ToString().Trim();
string font = "block";
string color = "33ff66";
string[] parts = text.Split(new char[] { ' ' }, StringSplitOptions.RemoveEmptyEntries);
var words = new System.Collections.Generic.List<string>();
foreach (string part in parts)
{
    if (part.StartsWith("font:", StringComparison.OrdinalIgnoreCase))
        font = part.Substring(5);
    else if (part.StartsWith("color:", StringComparison.OrdinalIgnoreCase) || part.StartsWith("colour:", StringComparison.OrdinalIgnoreCase))
        color = part.Substring(part.IndexOf(':') + 1);
    else
        words.Add(part);
}
string message = string.Join(" ", words.ToArray());
if (message.Equals("dismiss", StringComparison.OrdinalIgnoreCase))
{
    CPH.WebsocketBroadcastJson("{\"event\":\"ascii\",\"action\":\"dismiss\"}");
}
else
{
    if (message == "") message = "HELLO";
    font = System.Text.RegularExpressions.Regex.Replace(font, "[^a-zA-Z0-9_-]", "");
    color = System.Text.RegularExpressions.Regex.Replace(color, "[^a-zA-Z0-9#]", "");
    message = message.Replace("\\", "\\\\").Replace("\"", "\\\"").Replace("\r", "").Replace("\n", " ").Replace("|", "\\n");
    CPH.WebsocketBroadcastJson("{\"event\":\"ascii\",\"font\":\"" + font + "\",\"color\":\"" + color + "\",\"message\":\"" + message + "\"}");
}
```

### What to send

```json
{"event":"ascii","font":"block","color":"33ff66","message":"PIXEL\nFOLLOWED"}
```

| Field | Description |
| --- | --- |
| `event` | `ascii`. Required, so other overlays can share the server. |
| `message` | The text. `\n` starts a new line. `text` works too. |
| `font` | One of the names below. Omit it to use the URL default. |
| `color` | Hex (`33ff66` or `#33ff66`) or a name: `green`, `amber`, `gold`, `white`, `cyan`, `red`, `magenta`, `blue`, `orange`, `purple`. `colour` works too. |
| `hold` | Optional seconds to leave the message up. Default is 10. |
| `action` | `dismiss` clears the screen and the queue. |

The small pixel fonts (`block`, `big`, `slim`, `small`, `solid`, `star`) draw letters in uppercase. The banner fonts keep the case you send. Accented letters are folded (`José` becomes `Jose` or `JOSE`). Anything the font cannot draw becomes `?`.

## Fonts

Banner fonts use the same letter shapes as [TAAG](https://patorjk.com/software/taag/), including Graffiti. They are FIGlet fonts, kerned the way each font was designed.

| Name | Looks like | About how much fits |
| --- | --- | --- |
| `graffiti` | TAAG Graffiti. Alias `grafiti`. | 30 letters, 13 lines |
| `doom` | TAAG Doom | 45 letters, 10 lines |
| `slant` | TAAG Slant | 45 letters, 13 lines |
| `standard` | TAAG Standard | 45 letters, 13 lines |
| `speed` | TAAG Speed | 35 letters, 13 lines |
| `epic` | TAAG Epic | 30 letters, 9 lines |
| `figbig` | TAAG Big | 40 letters, 10 lines |
| `block` | `#` letters, 5×7. The default. | 53 letters, 12 lines |
| `big` | `#` letters, 7×9 | 40 letters, 9 lines |
| `slim` | `#` letters, 4×7 | 64 letters, 12 lines |
| `small` | `#` letters, 3×5 | 80 letters, 16 lines |
| `solid` | Filled cells, same shapes as `block` | 53 letters, 12 lines |
| `star` | `*` instead of `#`, same shapes as `block` | 53 letters, 12 lines |

Aliases: `large` is `big`, `narrow` is `slim`, `mini` is `small`, `dos` is `solid`, `grafiti` is `graffiti`.

Graffiti was designed by Leigh Purdie and fig-fonted by Leigh Purdie and Tim Maggio (1994). Doom is by Frans P. de Vries. Slant, Standard, and Big are by Glenn Chappell (Standard with Ian Chai). Speed and Epic are by Claude Martins. Doom, Slant, Standard, Big, Speed, and Epic include permission to modify them, with the modifier named in the font comments.

A line that is a little too long tightens the gap between the pixel letters. A single word that still cannot fit steps down to a narrower font. Words then wrap, and the whole block is centered on the 320×96 screen.

## URL parameters

| Parameter | Description |
| --- | --- |
| `preview` | `1` for a dark background, plus click / Space to test and Esc to dismiss. |
| `message` or `text` | Show this message as soon as the source loads. |
| `font` | Default font when a broadcast omits one. |
| `color` or `colour` | Default color when a broadcast omits one. |
| `hold` | Seconds the finished message stays up. Default `10`. |
| `scan` | Milliseconds per character cell while the cursor is receiving the screen. Default `0.625`, so one row still takes about a fifth of a second. `0` shows the message immediately. |
| `scroll` | Milliseconds between carriage returns while the message scrolls off. Default `80`. |
| `panel` | `1` draws a dark screen behind the 320×96 grid. |
| `auto` | `1` shows a sample alert as soon as the source loads. |
| `password` | WebSocket password, if authentication is enabled. |
| `host`, `port`, `endpoint` | WebSocket server. Default `127.0.0.1`, `8080`, `/`. |
| `debug` | `1` to log websocket events in the browser console. |
