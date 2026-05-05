# KiwiClient Wide Waterfall Support - Implementation Summary

## Changes Made

### 1. kiwi/client.py - `_set_wf_bins()` method
**Location:** Lines 426-429

**Before:**
```python
def _set_wf_bins(self, wf_bins):
    """Set the number of waterfall bins (frequency resolution)"""
    if wf_bins != 1024:
        self._send_message('SET wf_bins=%d' % wf_bins)
        self.WF_BINS = wf_bins
```

**After:**
```python
def _set_wf_bins(self, wf_bins):
    """Set the number of waterfall bins (frequency resolution)"""
    self._send_message('SET wf_bins=%d' % wf_bins)
    self.WF_BINS = wf_bins
```

**Reason:** Always send the command to the server, including the default 1024 value.

---

### 2. kiwi/client.py - `_process_wf()` method
**Location:** Lines 747-758

**Before:**
```python
def _process_wf(self, body):
    x_bin_server,flags_x_zoom_server,seq, = struct.unpack('<III', buffer(body[0:12]))
    data = body[12:]
    logging.info("W/F seq %d len %d x_bin_server %d" % (seq, len(data), x_bin_server))
    if self._options.netcat is True:
        return self._process_waterfall_samples_raw(seq, data)
    # ... rest of method ...
```

**After:**
```python
def _process_wf(self, body):
    x_bin_server,flags_x_zoom_server,seq, = struct.unpack('<III', buffer(body[0:12]))
    data = body[12:]
    
    # The actual number of bins is determined by the amount of data received
    # Each byte is one sample, so if we get 1024 bytes we have 1024 samples
    actual_bins = len(data)
    
    if seq == 0:  # Log on first packet only
        logging.info("W/F: requested %d bins, server sending %d bytes (%.0f bins)" % 
                    (self.WF_BINS, len(data), actual_bins))
    
    # Update WF_BINS to match what server is actually sending
    self.WF_BINS = actual_bins
    
    if self._options.netcat is True:
        return self._process_waterfall_samples_raw(seq, data)
    # ... rest of method ...
```

**Reason:** Automatically detect and adapt to the actual number of bins the server is sending, regardless of what we requested.

---

## How It Works

1. **Option Parsing (kiwirecorder.py):** `--wf-width=N` sets the requested bin count
2. **Initialization (kiwirecorder.py):** `KiwiWaterfallRecorder.__init__()` sets `self.WF_BINS = options.wf_width`
3. **Server Configuration (kiwirecorder.py):** `_setup_rx_params()` calls `self._set_wf_bins()`
4. **Send to Server (kiwi/client.py):** `_set_wf_bins()` sends `SET wf_bins=N` command
5. **Receive & Adapt (kiwi/client.py):** `_process_wf()` gets the actual data length and updates `self.WF_BINS` 
6. **PNG Generation (kiwirecorder.py):** PNG width = `len(samples)` which now matches actual server output

---

## PNG Width Behavior

The PNG width is determined by the actual amount of data received from the server:

- **Requested 2048, Server sends 1024:** PNG will be 1024 pixels wide
- **Requested 512, Server sends 1024:** PNG will be 1024 pixels wide  
- **Requested 1024, Server sends 1024:** PNG will be 1024 pixels wide

**Note:** Whether the server honors higher bin width requests depends on:
- Kiwi FPGA version and capabilities
- Whether it has the `SET wf_bins` command implemented
- Whether there's a hardware maximum (e.g., 1024 bins max)

The g3sdr.com server appears to only support 1024 bins based on testing.

---

## Testing

```bash
# Test with requested 512 bins
./kiwirecorder.py -s g3sdr.com -p 8075 --freq=147 \
  --wf --wf-png --wf-width=512 --log=info

# Result: PNG will be whatever the server actually sends
# (likely still 1024 on most servers due to FPGA limitations)
```

Check the log output: `W/F: requested 512 bins, server sending 1024 bytes`

---

## Future Improvements

If you have a KiwiSDR with a newer FPGA that supports variable bin widths:

1. Test with `--wf-width=2048` or higher
2. The code will automatically adapt and PNG width will scale appropriately
3. Consider detecting server capabilities via protocol negotiation

For manual testing with servers that DO support wide waterfalls:
```bash
./kiwirecorder.py -s <your-kiwi> --wf --wf-width=2048
# If server supports it, you'll get 2048-pixel-wide PNGs
```
