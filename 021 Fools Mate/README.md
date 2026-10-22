# Fools Mate Write-up

## 1. Enumeration

I started by scanning the target with Nmap:

```bash
nmap -sV -sC -oN scan 10.129.152.104
```

The scan showed that **port 80** was open and running HTTP.

I opened the website in my browser and found a chess game.

## 2. Chess Game

The objective was to find a **checkmate in one move**.

I found the correct move and played it, but instead of accepting the move, the website displayed a popup and rejected it.

This made me suspect that the move validation was being handled by client-side JavaScript.

## 3. JavaScript Analysis

I inspected the website using Chrome DevTools and found an `app.js` file containing the game logic.

I found the `preMoveCheck()` function:

```javascript
function preMoveCheck(from, to, promotion) {
    const probe = new Chess(game.fen());
    let result;

    try {
        result = probe.move({
            from,
            to,
            promotion: promotion || undefined
        });
    } catch (e) {
        result = null;
    }

    if (result && probe.isCheckmate()) {
        showSystemNotice("I'll shut down your PC if you play that.");
        return false;
    }

    return true;
}
```

The important part was the condition:

```javascript
if (result && probe.isCheckmate())
```

This condition always evaluates to `true`, so the function always executed the code that rejected the move.

## 4. Bypassing the Check

Since the validation was performed on the client side, I modified the JavaScript in Chrome DevTools.

I changed:

```javascript
if (result && probe.isCheckmate()) {
```

to:

```javascript
if (false) {
```

The modified function was:

```javascript
function preMoveCheck(from, to, promotion) {
    const probe = new Chess(game.fen());
    let result;

    try {
        result = probe.move({
            from,
            to,
            promotion: promotion || undefined
        });
    } catch (e) {
        result = null;
    }

    if (false) {
        showSystemNotice("I'll shut down your PC if you play that.");
        return false;
    }

    return true;
}
```

Now the rejection condition could never execute, so the function always reached:

```javascript
return true;
```

## 5. Getting the Flag

I played the correct checkmate-in-one move again.

This time, the move was accepted and I received the flag:

```text
THM{cl13nt_s1d3_ch3ckm4t3}
```

## 6. Key Takeaway

The main lesson from this room is that **client-side validation can be modified by the user**.

Important security decisions should not rely solely on JavaScript running in the browser because an attacker can inspect and modify the code before it executes.