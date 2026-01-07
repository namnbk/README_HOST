Symbol `rtcpReportsTimerId` not found

Searched for regex `RTCP|SenderReport` (`**/src/**/*.c`), 20 results

Searched for files matching `**/RtcpPacket.h`, 1 match

Read [](file:///d%3A/Users/Nam/amazon-kinesis-video-streams-webrtc-sdk-c/src/source/Rtcp/RtcpPacket.h#1-1), lines 1 to 50

Searched for files matching `**/*RtcpFunctionalityTest.cpp`, 1 match

The previous method of hijacking the RTP Timestamp is effective, but if you want other "practical" methods that do not involve modifying the SDK's C source code or hijacking protocols, here are your best alternatives:

### 1. Visual Latency Test (Gold Standard for QA)
This method measures the **True End-to-End Latency** (Glass-to-Glass), including camera capture delay, encoding, network, decoding, and screen rendering.

*   **How:**
    1.  Point the sender's camera at a computer screen.
    2.  On that screen, run a **high-precision stopwatch** (or a QR code generator that encodes the current time).
    3.  On the receiver side (browser), view the video stream.
    4.  Take a photo (or screenshot) capturing **both** the source screen (stopwatch) and the receiver screen (video stream) in the same frame.
*   **Calculation:**
    *   $t_{display} - t_{capture} = \text{Time on Source Screen} - \text{Time on Receiver Screen}$.
*   **Pros:** Requires **zero code changes**; accurate; measures everything.
*   **Cons:** Manual process; best for spot-checking, not real-time automated monitoring.

### 2. WebRTC Stats API (Software Estimation)
If you want an automated metric without modifying the sender, `chrome://webrtc-internals` or the JS `getStats()` API provides a built-in latency estimate.

*   **How:**
    The receiver (Browser) estimates latency using RTCP Sender Reports (which sync NTP time to RTP time).
    Use the following formula in your JavaScript viewer:
    
    $$Latency \approx \frac{\text{RTT}}{2} + \text{JitterBufferDelay} + \text{ProcessingDelay}$$

    You can get these from the `inbound-rtp` stats report:
    ```javascript
    const stats = await peerConnection.getStats();
    stats.forEach(report => {
        if (report.type === 'inbound-rtp' && report.kind === 'video') {
            // "jitterBufferDelay" is the total time spent in the jitter buffer
            // "jitterBufferEmittedCount" is the number of frames that left the buffer
            const avgBufferDelay = report.jitterBufferDelay / report.jitterBufferEmittedCount;
            
            // "roundTripTime" is available in the associated 'remote-inbound-rtp' or 'candidate-pair'
            // But often simpler to look at 'googCurrentDelayMs' if available (legacy) or calculate:
            
            console.log("Estimated Buffering + Processing:", avgBufferDelay, "seconds");
        }
    });
    ```
*   **Pros:** Built-in, no sender changes.
*   **Cons:** It is an **estimate**. It often misses the "Capture-to-Encode" latency on the sender side, which can be significant (30-100ms).

### 3. Audio/Visual Watermarking (Automated)
If you can overlay text on the video using GStreamer (which you are using), you can automate the "Visual Test".

*   **How:**
    1.  **Sender (GStreamer):** Use the `timeoverlay` element in your pipeline string. 
        *   Example: `... ! timeoverlay ! x264enc ...`
        *   This burns the timestamp directly into the pixels of the video frames.
    2.  **Receiver (JS):**
        *   Draw the `<video>` frame to a `<canvas>`.
        *   Use an OCR library (like `tesseract.js`) to read the text numbers from the canvas.
        *   Compare that read time `t_burned` with current wall clock `t_now`.
*   **Pros:** Works remotely; doesn't break protocols.
*   **Cons:** Heavy CPU usage on the receiver (OCR is slow); limited precision (usually per-frame, ~33ms).

**Recommendation:**
For a strictly "practical" engineering solution that balances accuracy and effort, **Method #1 (Visual Test)** is how 99% of video engineers validate latency during development. For production monitoring, **Method #2 (WebRTC Stats)** is the standard, even though it's an estimation. The "Timestamp Hijack" from the previous answer is the only way to get **precise per-frame software latency** without overlaying text on the video.
