# Waterfall Data Reception Analysis: OpenWebRX vs KiwiClient

## KEY INSIGHT: How OpenWebRX Receives Waterfall Data

### OpenWebRX's Approach (openwebrxraw.js)

**Critical code pattern:**
```javascript
function on_ws_recv(evt, ws) {
    var data = evt.data;
    if (!(data instanceof ArrayBuffer)) {
        console.log("on_ws_recv: not an ArrayBuffer?");
        return;
    }
    
    // Extract the first 3 characters to determine message type
    var firstChars = arrayBufferToStringLen(data, 3);
    
    if (firstChars == "CLI") 
        return;  // Ignore keepalive
    
    var claimed = false;
    
    if (firstChars == "MSG") {
        // TEXT MESSAGE - parse key-value pairs
        var stringData = arrayBufferToString(data);
        params = stringData.substring(4).split(" ");
        // Process as text messages
        
    } else {
        // BINARY DATA - pass directly to recv_cb for waterfall/audio
        if (ws.recv_cb && (ws.stream != 'EXT' || kiwi_flush_recv_input == false)) {
            ws.recv_cb(data, ws, firstChars);  // Binary data callback!
        }
    }
}
```

**Key setup for W/F stream:**
```javascript
kmap.wf_ws = open_websocket('W/F', function() {
    // ... send configuration ...
    kmap.wf_ws.send("SET zoom=" + zoom_level + " start=" + x_bin);
    kmap.wf_ws.send("SET maxdb=0 mindb=-100");
    kmap.wf_ws.send("SET wf_speed=3");
}, kmap, null, 
   kiwi_map_waterfall_add_queue,  // ← Binary data callback!
   kiwi_map_wf_preview_error_cb, 
   kiwi_map_waterfall_close, {...}
);

function recv_websocket(ws, recv_cb) {
    if (ws.stream == 'SND' || ws.stream == 'W/F') 
        return;  // ← W/F streams never use recv_cb!
    ws.recv_cb = recv_cb;
}
```

---

## KiwiClient's Current Approach (kiwi/client.py)

**Message processing:**
```python
def _process_message(self, tag, body):
    if tag == 'MSG':
        self._process_msg(bytearray2str(body[1:]))
    elif tag == 'SND':
        self._process_aud(body)
    elif tag == 'W/F':
        self._process_wf(body[1:])  # ← Binary waterfall data
    elif tag == 'EXT':
        body = bytearray2str(body[1:])
        for pair in body.split(' '):
            # ... parse key-value pairs ...
```

---

## THE PROBLEM & THE SOLUTION

### Why PNG is Always 1024 Pixels Wide

The issue is **NOT** just about client-side variables or PNG writing code. The real flow is:

1. **Server** receives `SET wf_bins=2048` command
2. **Server** sends waterfall data with 2048 frequency bins
3. **Client** receives binary waterfall data in `W/F` stream
4. **Client** parses the 12-byte header: `x_bin_server, flags_x_zoom_server, seq`
5. **Client** reads `x_bin_server` - this tells what bin count the server is actually sending!

**Current bug:** KiwiClient sets `self.WF_BINS = 1024` in parent class and only overwrites it when `--wf-width=N` is passed, but:
- The `_set_wf_bins()` method now sends the command ✓
- BUT: The client never **reads** `x_bin_server` from the waterfall header to confirm what the server is actually sending
- It just assumes it's getting 1024 and creates 1024-pixel-wide PNG

### What OpenWebRX Does Differently

OpenWebRX's waterfall callback (`kiwi_map_waterfall_add_queue`) receives **raw binary data** and must parse:
- The 12-byte header to extract `x_bin_server` 
- This tells it how many bins the server is actually sending
- Then it dynamically sizes its canvas/rendering based on actual data

### The Fix Required

In `kiwi/client.py`, in `_process_wf()` method:

```python
def _process_wf(self, body):
    # Parse the 12-byte waterfall header
    x_bin_server, flags_x_zoom_server, seq = struct.unpack('<III', body[0:12])
    
    # UPDATE: Use server's actual bin count, not our local WF_BINS
    actual_bins = x_bin_server
    
    logging.info("W/F seq %d len %d x_bin_server %d (expected %d)" % 
                 (seq, len(body), x_bin_server, self.WF_BINS))
    
    # If server didn't honor our request, adjust
    if actual_bins != self.WF_BINS:
        logging.warning("Server sent %d bins but we requested %d" % (actual_bins, self.WF_BINS))
        self.WF_BINS = actual_bins  # ← Use what server actually sent
    
    # Continue with rest of waterfall processing...
```

---

## KEY DIFFERENCES

| Aspect | OpenWebRX | KiwiClient |
|--------|-----------|-----------|
| **Stream Setup** | Dedicated 'W/F' stream with recv_cb | Uses same message processing |
| **Binary vs Text** | Distinguishes by first 3 bytes | Tags messages as 'W/F' or 'MSG' |
| **Bin Count** | Reads from `x_bin_server` in every packet | Assumes fixed value |
| **Scaling** | Dynamic based on actual received data | Fixed to WF_BINS variable |
| **PNG Width** | Scales to match actual received bins | Hardcoded to len(samples) // 3 |

---

## RECOMMENDATION

The fix is to add code to actually **read and use** the `x_bin_server` value from the waterfall header. This value indicates what bin width the server is actually sending, not what we requested.

**Step-by-step fix:**
1. ✓ Already done: Send `SET wf_bins=N` command to server
2. ✓ Already done: Set `self.WF_BINS = N` on client
3. **TODO:** Read `x_bin_server` from each waterfall packet
4. **TODO:** Verify server honored the request (x_bin_server == requested)
5. **TODO:** Handle case where server can't/won't send requested bin count
6. **TODO:** PNG width will automatically scale once `self.WF_BINS` reflects actual received data
