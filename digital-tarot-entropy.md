# Digital tarot draws: personal entropy vs. secure randomness

Notes compiled 2026-10-07.

## 1. The question

A CSPRNG gives a uniformly random card but feels disconnected from the querent. What, if anything, makes a draw "yours", and how can an app reproduce that?

## 2. Occultist positions

Three camps, each implying a different mechanism.

### Camp 1 — Synchronicity: the source doesn't matter, the involvement does
- Jungian view: the card is not *caused* by the subconscious, it is *meaningfully coincident* with it. Jung built synchronicity on I Ching coin tosses, which don't "know" the querent either.
- On this view a CSPRNG is as valid as a paper deck.
- Proponents: Mary K. Greer; Colette Baron-Reid (says she enjoys digital versions of her decks as much as physical ones); Fool's Dog developers (Caroline Kenner: the interface "opens the door to synchronicity"; two electronic shuffle modes plus deck cuts to add random factors).

### Camp 2 — Traditionalist: the hand must touch the process
- Witchcraft / professional-reader side of the Vice / Wild Hunt debate: divination should be done by humans, not apps (Blue June); technology only as an aide (Maria Palma-Drexler); apps rely on algorithms instead of hand shuffling, harming the random nature of the pull (Mat Auryn, though he conceded apps are not useless).
- Physical correlate: "think around the subject as you shuffle and stop when you feel finished" — with the caveat that concentrating too hard pushes one's desires into the shuffle.
- The "connection" here is the **stopping moment** and the **choice from a fan**, not the entropy of the shuffle itself.

### Camp 3 — Mind affects the random source
- Parapsychology / PEAR-style claim: intention can bias random number generators; hardware/quantum sources preferred over pseudo-RNG.
- Randonautica built on this: quantum RNG + Jungian synchronicity.
- Implication: a physical noise source sampled *while* the user holds intent, not a seeded algorithm computed in a microsecond.

### Design mechanisms, ordered by how much "you" is in the draw
1. Hardware/quantum RNG sampled during the user's act (hold → release). Satisfies camp 3; harmless to 1–2.
2. Cascade loop / stop-the-shuffle: cards stream until a tap; the stop doubles as intention-setting. Fan-and-drag variant: drag across a face-down arc and stop on one. Shake-to-shuffle, stop-shaking-to-reveal is the older form.
3. Body as entropy: accelerometer/gyro, touch timing and pressure, mic noise, mixed into the seed. Closest digital analogue to hand-shuffling.
4. Virtual cut: user chooses where to cut the shuffled stack.
5. Transparency: commit (hash) to the deck order before the pick so the app provably didn't reorder afterwards; never reweight cards for engagement.

Combination meeting every camp's criterion: hardware noise seeds → user gesture perturbs the seed → user stops the cascade → user cuts → user picks from a fan.

## 3. Personal vs. universal entropy

**Personal (caused by the body)**
- Accelerometer/gyro during shake or hold — richest source (~60 Hz, many bits/s, unrepeatable).
- Touch stream: coordinates, pressure/radius, velocity, inter-event timing while dragging or rubbing.
- Stop moment: ms timestamp of the ending tap (~10 bits; the act closest to "stop when it feels finished").
- Microphone noise — personal-ish but ambient; heavier permission.

**Universal (external to the user)**
- `crypto.getRandomValues()` — excellent randomness, nothing of the user in it.
- Quantum/hardware RNG services — "universal" in the strongest sense.

Ordered most → least personal: stop-tap timing ≈ touch-drag stream > motion sensors (most bits, fully yours) > ambient mic > OS CSPRNG > remote quantum RNG. The cryptographically best source is the least personal.

Pure hand-shuffle analogue: `deck = hash(motion ‖ touch ‖ stop timestamp)`, no external source. Risk: a still phone yields little entropy → enforce a minimum collection (section 5).

Compromise: hash personal and external sources together. Guarantees uniformity; arguably faithful, since a paper deck's order also depends on the order it was in before you shuffled. A time component (the stop timestamp) already removes the theoretical "identical gesture → identical card" case.

## 4. iOS web availability of motion sensors

- `DeviceMotionEvent` (acceleration, accelerationIncludingGravity, rotationRate, interval) and `DeviceOrientationEvent` (alpha/beta/gamma) are available in Safari ≥ 17; "widely available" baseline since March 2026.
- **HTTPS only** — iOS auto-declines on non-secure pages.
- **Permission from a user gesture**: `DeviceMotionEvent.requestPermission()` must be called inside a tap/click handler (iOS 13+); Safari shows a one-time prompt. Per-origin; resettable in Settings → Safari → Motion & Orientation Access.
- Generic Sensor API (`new Accelerometer()`) is not in Safari — use the event API.
- Works in Safari, iOS WebViews and home-screen PWAs.

```js
drawButton.addEventListener('click', async () => {
  if (typeof DeviceMotionEvent.requestPermission === 'function') {
    const state = await DeviceMotionEvent.requestPermission();
    if (state !== 'granted') return fallbackToTouchEntropy();
  }
  window.addEventListener('devicemotion', e => {
    entropy.mix(e.accelerationIncludingGravity, e.rotationRate, e.interval);
  });
});
```

The "Draw" tap doubles as the permission gesture. Keep a touch-only fallback for users who decline.

## 5. Enforcing a minimum entropy collection ("shake the mouse" gate)

Accumulate all raw samples into a hash; keep a *conservative* running credit of bits; unlock the stop button only when credit ≥ target.

**Targets**
- 78! ≈ 2^391 orderings; one card ≈ 6.3 bits; 10-card spread with reversals ≈ 70 bits.
- Aim for 128–256 credited bits → a few seconds of shaking, roughly the ritual length wanted anyway.

**Crediting rules** (credit only the unpredictable part; mix everything regardless)
- Motion: per `devicemotion` event, delta from previous sample per axis; credit ~0.7 bit per axis whose |Δ| exceeds a noise floor; 0 when resting.
- Touch/drag: ~2–3 bits per pointer event from Δt jitter and Δx/Δy; 0 when still.
- Stop tap: ~8 bits from the sub-second timestamp.
- Cap credit per time window (e.g. 32 bits / 100 ms) so a violent shake cannot finish the gate faster than N seconds.

**Sketch**

```js
const TARGET_BITS = 192, MAX_BITS_PER_100MS = 32, NOISE = 0.15;
let credited = 0, windowBits = 0, last = null, buf = [];

function mix(...nums) { buf.push(...nums, performance.now()); }

function credit(bits) {
  const room = MAX_BITS_PER_100MS - windowBits;
  const b = Math.min(bits, Math.max(0, room));
  windowBits += b; credited += b;
  ui.progress(Math.min(1, credited / TARGET_BITS));
  if (credited >= TARGET_BITS) ui.enableStop();
}
setInterval(() => (windowBits = 0), 100);

window.addEventListener('devicemotion', e => {
  const a = e.accelerationIncludingGravity, r = e.rotationRate;
  const v = [a.x, a.y, a.z, r.alpha, r.beta, r.gamma];
  mix(...v);
  if (last) {
    const moving = v.filter((x, i) => Math.abs(x - last[i]) > NOISE).length;
    credit(moving * 0.7);
  }
  last = v;
});

canvas.addEventListener('pointermove', e => {
  mix(e.clientX, e.clientY, e.pressure, e.width);
  credit(2);
});

stopBtn.onclick = async () => {
  mix(performance.now());
  credit(8);
  const bytes = new TextEncoder().encode(buf.join(','));
  const seed = await crypto.subtle.digest('SHA-256', bytes);
  deck = fisherYates(cards, drbgFrom(seed));
};
```

**Refinements**
- Drive Fisher–Yates with a proper DRBG (HMAC-DRBG or ChaCha keyed by the SHA-256 seed), never `Math.random`, so deck order is a pure function of the gestures.
- If mixing in `crypto.getRandomValues()`, add it as a final hash input and do **not** credit it — the progress bar measures only the user's contribution.
- Stop button visibly disabled until the bar fills: the digital "shuffle until it feels done", with an enforced floor. Touch-only fallback still fills the bar, just slower.

## Sources

- [Column: Divination on the Download (The Wild Hunt)](https://wildhunt.org/?p=17733)
- [Putting divination first: Fool's Dog tarot apps (The Wild Hunt)](https://wildhunt.org/2016/09/putting-divination-first-fools-dog-tarot-apps.html)
- [Digital card decks yes or no (The Augurly)](https://theaugurly.substack.com/p/digital-card-decks-yes-or-no)
- [How to shuffle Tarot cards properly](https://voices.girleffect.org/how-to-shuffle-tarot-cards-properly.html)
- [A replication of the slight effect of human thought on a pseudorandom number generator](https://irepository.uniten.edu.my/handle/123456789/30099)
- [Randonautica, synchronicity and the dérive (Radical Art Review)](https://www.radicalartreview.org/post/acid-left-review-radionautica-synchronicity-and-the-situationist-dérive)
- [Tarot card shuffle animation design notes](https://vp0.com/blogs/tarot-card-shuffle-animation-swiftui.md)
- [Interpretive Cultures: resonance, randomness and AI-assisted tarot (arXiv)](https://arxiv.org/pdf/2602.11367)
- [MDN — Device orientation events](https://developer.mozilla.org/de/docs/Web/API/Device_Orientation_Events)
- [web-features: device orientation events support data](https://web-platform-dx.github.io/web-features-explorer/features/device-orientation-events.json)
- [Capacitor Motion docs (iOS permission pattern)](https://capacitorjs.com/docs/next/apis/motion)
- [HTML5 accelerometer and iOS — HTTPS requirement](https://community.openfl.org/t/html5-accelerometer-and-ios/12773)
