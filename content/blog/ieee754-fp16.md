+++
title = "The Curious Case of Floating Point Numbers"
date = "2026-06-06"
description = "An introduction to the IEEE-754 floating point standard."
tags = ["mathematics"]
+++

## The Bug That Isn't

You've probably seen this. You write `0.1 + 0.2` in Python and expect `0.3`. Instead, you get `0.30000000000000004`. Your first instinct: "This is a bug. Computers are broken." Your second instinct: "I'll use Decimal." Your third instinct: "Actually, let me understand what's happening."

What's happening is far more interesting than a bug. It's a fundamental mismatch between how humans think about numbers (decimal, infinite precision) and how computers store them (binary, finite bits). This mystery leads us into floating-point representation[^2], the standard that governs nearly every floating-point calculation on earth (and beyond) — plus it's surprisingly elegant once you know the rules.

## Following the Binary Trail

Here's the core problem: your computer can't count in decimal. It only counts in binary (0s and 1s). When you write down `0.1` in decimal, you're saying "one tenth." But what is one tenth in binary?

$$0.1_{\text{decimal}} = \frac{1}{10}$$

Try to express this as a sum of powers of 2:

$$\frac{1}{10} = \frac{?}{2} + \frac{?}{4} + \frac{?}{8} + \frac{?}{16} + ...$$

You can't. Not exactly. In binary, $0.1$ is a repeating fraction: $0.0\overline{0011}$ (the digits 0011 repeat forever). It's like trying to write $\frac{1}{3}$ in decimal—you get $0.333...$, and you can't stop.

When you store $0.1$ in a computer, you must round it. The rounding happens silently, every time. When you add two rounded numbers, the errors compound. That's why `0.1 + 0.2` doesn't equal `0.3`.

It's not malice. It's math.

## The IEEE-754 Rulebook

In 1985, a group of computer scientists and engineers said: "We need a standard for floating-point numbers that works everywhere." They created IEEE-754[^1], and it's one of the most widely popular standards in computing. Almost every device—your phone, your graphics card, your cloud server—uses IEEE-754.

The standard says: represent every number in this form:

$$x = (-1)^s \times 2^{e - \text{bias}} \times (1 + f)$$

Breaking it down:
- $s$ is the sign bit (0 for positive, 1 for negative). Simple.
- $e$ is the exponent bits (the raw value stored). We subtract a **bias** to get the actual exponent.
- $f$ is the fractional part, also called the mantissa or significand. It's the "meat" of the number.

### The Role of Bias: A Clever Trick

Here's where the design gets clever. Instead of dedicating a sign bit to the exponent (so we could represent both positive and negative exponents), IEEE-754 uses a **bias**—an offset. Why? Because it allows unsigned comparison. If you compare two floating-point bit patterns directly, you can tell which number is larger without any decoding. Hardware loves this trick.

Here's how it works:

$$\text{Actual exponent} = e_{\text{stored}} - \text{bias}$$

For each format, the bias is chosen to be roughly half of the maximum storable exponent:

| Format | Exponent Bits | Max Exponent | Bias | Range |
|--------|---------------|--------------|------|-------|
| FP16   | 5             | 31           | 15   | $-14$ to $+15$ |
| FP32   | 8             | 255          | 127  | $-126$ to $+127$ |
| FP64   | 11            | 2047         | 1023 | $-1022$ to $+1023$ |

**Example**: In FP16, if the exponent bits store `01111` (which is 15 in decimal), the actual exponent is $15 - 15 = 0$. So the number is $2^0 = 1.something$.

If the exponent bits store `10000` (16 in decimal), the actual exponent is $16 - 15 = 1$. So the number is $2^1 = 2.something$.

If you want to represent very small numbers, you'd store `00001`, giving $1 - 15 = -14$, so $2^{-14} \approx 6.1 \times 10^{-5}$.

This is why FP16 can represent numbers from roughly $10^{-4}$ to $10^{4}$—the bias shifts the exponent range to include both large and small values symmetrically. Clever.

### The Three Suspects: Format Tradeoffs

Different formats trade off precision for speed and size:

| Format | Bits | Sign | Exponent | Fraction | Bias | Precision (bits) | Range |
|--------|------|------|----------|----------|------|------------------|-------|
| FP16   | 16   | 1    | 5        | 10       | 15   | 11               | ~$10^{±4}$ |
| FP32   | 32   | 1    | 8        | 23       | 127  | 24               | ~$10^{±38}$ |
| FP64   | 64   | 1    | 11       | 52       | 1023 | 53               | ~$10^{±308}$ |

**FP16** is the speedster—a suspect with motive (speed) and means (compact). It fits in 2 bytes, locks into GPU registers, and runs at blazing speed. Perfect for machine learning and graphics where fractional accuracy is negotiable. But it only gives you about 3 decimal digits. That's enough for "is this pixel brighter?" but not for accounting.

**FP32** is the reliable suspect—the one everyone trusts. It picks all three: precision (~7 digits), speed (native on most hardware), and reasonable size (4 bytes). Most of the world's scientific and engineering code runs on FP32. The default choice for good reason.

**FP64** is the paranoid suspect—checks everything twice. 8 bytes, 15 decimal digits, range spanning $10^{±308}$. For climate simulations, financial modeling, and anywhere errors compound over time, FP64 is the insurance policy.

## Examining the Evidence: A Floating-Point Number

Let's get our hands dirty and decode an actual FP16 number. Take $1.5$:

$$1.5 = 1 + 0.5 = 1.1_{\text{binary}}$$

In IEEE-754 FP16:
- **Sign** ($s$): $0$ (it's positive)
- **Exponent** ($e$): The number is $1.1_{\text{binary}} \times 2^0$, so the exponent is $0$. Stored with bias: $0 + 15 = 15$.
- **Fraction** ($f$): The implicit leading 1 gives us the "1" part. The fraction bits store $.1_{\text{binary}} = 0.5$, but represented without the leading 1, so it's just 10 zeros followed by... wait, actually, let me redo this.

Actually: $1.5 = 1.1_{\text{binary}} = 1.\underbrace{1000000000}_{\text{10 bits}}$ (in normalized form, it's 1 point something).

So:
- Sign: `0`
- Exponent: `01111` (that's 15 in binary)
- Fraction: `1000000000` (the first bit is 1, the rest are 0)

Full binary: `0 01111 1000000000`

In hexadecimal: `0x3C00`

That's just 4 hex digits to represent 1.5. Not bad.

## Reconstructing the Crime Scene

Let's trace a nastier example: $-0.125$ in FP16. Negative, and a fraction. Let's break it down.

$$-0.125 = -\frac{1}{8} = -2^{-3} = -1.0_{\text{binary}} \times 2^{-3}$$

In IEEE-754 FP16:
- **Sign**: `1` (negative)
- **Exponent**: $-3 + 15 = 12$. In binary: `01100`.
- **Fraction**: The mantissa is exactly $1.0$, so the fraction bits are all zeros: `0000000000`.

Full binary: `1 01100 0000000000` = `0xB000` in hex.

Now here's the thing: this representation is *exact*. There's no rounding error. We can represent $-0.125$ perfectly in FP16 because it's a power of 2. But try to represent $0.1$ and you hit the repeating fraction problem again. The format can only represent certain numbers exactly. Everything else is an approximation.

## The Missing Digits

Here's where FP16 starts to sweat. It has only 10 fraction bits (11 with the implicit leading 1), which means it can represent about **3 to 4 decimal digits** of precision. That sounds bad until you realize: it's still amazing that you fit that much into 2 bytes.

Let's compare how three formats handle $\pi$:

$$\pi \approx 3.14159265358979323846...$$

- **FP16**: $3.140625$ (error: $\approx 0.0009$ or about 0.03%)
- **FP32**: $3.14159274$ (error: $\approx 10^{-7}$ or about 0.000003%)
- **FP64**: $3.14159265358979323846...$ (error: $\approx 10^{-16}$ – essentially perfect)

FP16 is *rough*. If you're using it to represent the radius of Earth, you'll be off by a few meters. For ML, that's fine—the network adjusts. For navigation, not so much.

FP32 is solid. Most graphics you see—pixels shaded, lighting calculated, transformations applied—all run on FP32. It's precise enough that most people never notice an error.

FP64 is overkill for most tasks but necessary for some. Climate simulations, long-running integrations, financial calculations where compound errors matter.

## The Culprit Revealed

The culprit isn't malice—it's the finite representation. A computer can't store infinite digits. It has to round. And rounding is where all the trouble starts.

IEEE-754 specifies exactly *how* rounding happens, which is good—it means the behavior is predictable. But the error is still there, waiting.

For FP16, the smallest positive normal number is $2^{-14} \approx 6.1 \times 10^{-5}$. Try to store anything smaller than that, and you hit a problem: the exponent bits are at their minimum. The format has a fail-safe: **subnormal numbers**. They're represented with an exponent of all zeros and a fraction that doesn't have the implicit leading 1. They're slower, less precise, but they prevent sudden drops to zero.

Below the subnormal minimum, you simply underflow to zero. It's a silent failure: the number you tried to store just... disappears. In FP16, this happens around $10^{-8}$—tiny, but real.

The same problem exists on the other end. The largest FP16 number is about $65500$. Go above that and you overflow to infinity. Your result becomes `inf`, which propagates through further calculations like a contagion.

## The Shady Abettors: Infinity and NaN

IEEE-754 reserves special bit patterns for edge cases. These aren't errors—they're deliberate design decisions to handle situations gracefully.

**Infinity** (`+inf` and `-inf`): When you overflow (the number is too big to represent), you get infinity. When you divide by zero, you also get infinity (with the sign of the numerator). Infinity has special arithmetic: $\infty + 1 = \infty$, $\infty / \infty = \text{NaN}$. It's contagious—once you have it, it propagates through calculations.

**NaN** (Not a Number): Represents an undefined result. The weird part: `NaN != NaN`. Comparison always returns false. If you ever get a NaN, your debug checks with `if (x == NaN)` will fail. Use `isnan(x)` instead.

**Subnormal numbers**: Exponent all 0s, fraction non-zero. These represent numbers smaller than the normal minimum. They sacrifice precision to avoid sudden underflow to zero. In FP16, they extend the range down to about $10^{-8}$ instead of $10^{-5}$, but with fewer significant bits. They're slower on some hardware.

**Signed zero**: Yes, `+0` and `-0` exist. They compare equal but have different bit patterns. This matters in complex math: $1 / (+0) = +\infty$ while $1 / (-0) = -\infty$.

This table summarizes the special values:

| Value | Exponent | Fraction | Meaning | Example |
|-------|----------|----------|---------|---------|
| $+\infty$ | All 1s | All 0s | Positive overflow | `1.0 / 0.0`, overflow |
| $-\infty$ | All 1s | All 0s | Negative overflow | `-1.0 / 0.0`, overflow |
| NaN | All 1s | Non-zero | Undefined result | `0.0 / 0.0`, $\sqrt{-1}$ |
| +0 | All 0s | All 0s | Positive zero | `0.0` |
| -0 | All 0s | All 0s | Negative zero | `-0.0` |
| Subnormal | All 0s | Non-zero | Numbers $< 2^{\text{-bias}+1}$ | Very small numbers |

## Judgment Day: Rounding Rules

When you compute something like `0.1 + 0.2`, the result doesn't fit perfectly in FP32 or FP16. The hardware has to round. And *how* it rounds matters.

IEEE-754 defines five rounding modes:

| Mode | Name | Example: 2.5 rounded to int | Example: 3.5 rounded to int | Use Case |
|------|------|----------------------------|----------------------------|----------|
| 0 | Round to nearest, ties to even | 2 (even) | 4 (even) | Default, statistical computing |
| 1 | Round toward zero | 2 | 3 | Truncation, embedded systems |
| 2 | Round toward $+\infty$ | 3 | 4 | Interval arithmetic, upper bounds |
| 3 | Round toward $-\infty$ | 2 | 3 | Interval arithmetic, lower bounds |
| 4 | Round away from zero | 3 | 4 | Rare, legacy systems |

**Round to nearest, ties to even** (default): The star witness. It rounds to the nearest representable value. When you're exactly between two values (a "tie"), it rounds toward the even bit—called "banker's rounding." It's brilliant because errors cancel instead of accumulating. Over millions of operations, the tiny rounding biases average out.

**Round toward zero**: Just truncate. Simple, but biased—always underestimates magnitude.

**Round toward $\pm\infty$**: Used in interval arithmetic. "Round down" keeps the lower bound safe; "round up" keeps the upper bound safe. If you need guaranteed bounds, these modes are your alibi.

**Round away from zero**: Rarely used. Some legacy systems and old compilers.

This is why statistical code tends to work well even with floating-point—the default rounding mode is designed to minimize cumulative bias.

## Real Victims of Floating-Point Errors

Now let's talk about the real casualties. This is where floating-point bites.

**Catastrophic cancellation**: The big one. Imagine you compute $(x + \epsilon) - x$ where $\epsilon$ is tiny. Mathematically, you get $\epsilon$. But in floating-point, if $x \gg \epsilon$, the fractional bits of $x + \epsilon$ and $x$ are identical. When you subtract, you get garbage or zero. In FP16, this happens fast because precision is so low.

A real case: the quadratic formula $x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}$. If $b^2 \gg 4ac$, then $\sqrt{b^2 - 4ac} \approx |b|$, and you're subtracting nearly equal numbers: $-b \pm \sqrt{...}$. Disaster. You lose significant digits. Numerical analysts use the rearranged formula to dodge this.

**Summation errors**: Add a million tiny numbers into a large sum, and the tiny ones disappear. The exponent adjusts to the large number's scale, and the fractional bits don't have room for small contributions. Sum smallest to largest (or use Kahan summation) to keep them.

**Division amplifies errors**: Divide by a number close to zero, and small rounding errors in the numerator explode. This is why direct matrix inversion fails in linear algebra—you use LU decomposition instead.

**Oscillation**: With FP16's low precision, repeated operations can cause wild oscillations. Rounding at each step pushes you above or below thresholds, creating unexpected jumps.

## Lessons for Future Investigations

We've unraveled the mystery. Now here's your investigator's handbook—rules for surviving in the floating-point world:

1. **Never use `==` to compare floats**. Use a tolerance instead: `abs(a - b) < epsilon`. The epsilon depends on scale. For numbers around 1, use `1e-6`. For large numbers, use relative tolerance: `abs(a - b) / abs(a) < 1e-6`.

2. **Know your format's precision limits**. FP16: ~3 decimal digits. FP32: ~7. FP64: ~15. Plan accordingly. Don't expect FP16 to give you accuracy it can't possibly deliver.

3. **Watch for catastrophic cancellation**. If you're subtracting nearly equal numbers, rearrange the math algebraically first. The quadratic formula example is your template.

4. **Sum carefully**. Smallest to largest (or Kahan summation). For critical work, accumulate in higher precision, then downcast.

5. **Test edge cases**. Very large, very small, very close to zero, extreme exponents. IEEE-754 handles them, but your code might not.

6. **Choose the right format for the job**. FP16 for graphics and ML where speed dominates. FP32 for general science. FP64 for simulations and financial calculations where errors compound.

## Closing the Case

IEEE-754 is not a bug. It's a carefully negotiated tradeoff. You can't represent infinite precision in finite bits. You choose: speed, size, accuracy. Pick two. That's the fundamental deal.

**FP16** trades accuracy for speed and space. It's everywhere because GPUs are fast and neural networks don't care about 3% errors. A pixel off by 3% is invisible. Your bank balance off by 3% is not.

**FP32** is the middle ground. Most of the scientific and engineering world runs on it. Precise enough, fast enough, reasonable size. The default choice.

**FP64** is the insurance policy. For climate models, long simulations, financial systems where errors compound. When precision matters more than everything else.

The real lesson: understand your tools. Floating-point isn't broken—you just have to respect its limits. Write code that checks for NaN, handles overflow, picks the right precision, and avoids the numerical pitfalls. Do that, and IEEE-754 will serve you reliably for decades.

**The case is closed.** The mystery of `0.1 + 0.2 ≠ 0.3` is solved. It wasn't a bug. It was the rules of the game, written in 1985 and unchanged because they work.

---

[^1]: IEEE Standard 754-2019 for Floating-Point Arithmetic. Official standard document: https://ieeexplore.ieee.org/document/8766229

[^2]: Wikipedia entry on Floating-Point Arithmetic: https://en.wikipedia.org/wiki/Floating-point_arithmetic
