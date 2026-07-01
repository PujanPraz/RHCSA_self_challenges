# 🐧 RHCSA Hands-On Self Challenge — Week 2 Part 3 Notes

> **Topic:** Swap Space
> **Environment:** CentOS 10 (VirtualBox VM)

---

## 📋 Table of Contents

- [What is Swap Space?](#what-is-swap-space)
- [Day 13 — Creating a Swap File](#day-13--creating-a-swap-file)
- [Quick Reference Card](#quick-reference-card)
- [Interview Questions](#interview-questions)

---

## What is Swap Space?

### The Concept

RAM is fast but limited. When applications try to use more memory than available RAM, the system would normally crash.

**Swap space is emergency overflow space on disk that acts like extra RAM.**

> Think of RAM like your office desk. You can only fit so many papers on it. When the desk is full, you put extra papers in a drawer — slower to access, but at least you don't lose them. That drawer is swap.

```
RAM full → Linux moves idle/inactive data to swap → frees up RAM → system stays alive
```

### Important Truth About Swap

Swap is **much slower** than RAM because it lives on disk, not memory chips. It is NOT meant to be your main memory — it exists to prevent crashes, not to make the system faster.

> Swap is a parachute, not a better engine. It saves you from crashing, but it won't make you fly faster.

### Two Ways to Create Swap

| Method | When to use |
|--------|-------------|
| Swap partition | Set up during OS installation, fixed size |
| Swap file | Can create anytime, flexible, easier to resize — most common in real world |

---

## Day 13 — Creating a Swap File

### 🎯 Scenario
> A server running memory-heavy applications has limited RAM. Create a 2GB swap file to give it breathing room. Make it survive reboots.

---

### 📖 Concepts

#### Checking Current Memory Status

```bash
free -h
```

```
               total    used    free    available
Mem:           3.5Gi    1.3Gi   917Mi   2.3Gi
Swap:          2.0Gi      0B    2.0Gi
```

| Row | Meaning |
|-----|---------|
| `Mem` | Actual RAM — total, used, free, available |
| `Swap` | Swap space — total, used, free |

> If swap `used` is constantly high, that's a warning sign the server needs more RAM — it keeps running out and relying on slow disk overflow.

#### What is fallocate?

`fallocate` instantly reserves disk space for a file, without writing data byte by byte.

**Old slow way:**
```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048
```
Actually writes 2GB of zeros to disk — slow for large files.

**Fast way:**
```bash
sudo fallocate -l 2G /swapfile
```
Instantly reserves 2GB at the filesystem level.

> Think of it like reserving a restaurant table. You don't need someone to physically sit in every chair — you just tell the host "reserve this for 2 hours" and it's instantly blocked off.

#### Important: /swapfile is a FILE, not a folder

```bash
-rw-------. 1 root root 2147483648 Jul 1 07:32 /swapfile
        ↑
   no 'd' at the start = regular file, not a directory
```

It's one single 2GB file. Linux uses the raw space inside it as swap memory — it's not something you can open and browse like a folder.

#### Why chmod 600?

Swap can temporarily hold sensitive data copied out of RAM. Only root should be able to read or write to the swap file.

```
rw-r--r--  ← BEFORE: everyone can read (security risk!)
rw-------  ← AFTER: only root can read/write ✅
```

---

### 💻 Commands

```bash
# Step 1 — Check current memory
free -h

# Step 2 — Check available disk space
df -h /

# Step 3 — Create the swap file (instantly reserves 2GB)
sudo fallocate -l 2G /swapfile

# Step 4 — Secure it — root only
sudo chmod 600 /swapfile

# Step 5 — Verify permissions
ls -lh /swapfile
# -rw-------. 1 root root 2.0G Jul 1 07:33 /swapfile

# Step 6 — Format it as swap
sudo mkswap /swapfile
# Output: Setting up swapspace version 1, size = 2 GiB

# Step 7 — Activate it temporarily
sudo swapon /swapfile

# Step 8 — Verify
free -h
# Swap: 4.0Gi (2GB original LVM swap + 2GB new swapfile)
```

### ⚠️ Temporary Only — Test It

```bash
sudo reboot
free -h
# Swap: 2.0Gi ← back to original! Swapfile didn't survive reboot
```

Just like disk mounts before fstab — swap needs a permanent entry too.

---

### Making It Permanent with fstab

```bash
sudo nano /etc/fstab
```

Add this line:

```
/swapfile    none    swap    defaults    0 0
```

> Notice: swap doesn't need a real mount point since it's not a folder you browse — that's why the second column is `none`.

```bash
# Test without rebooting
sudo swapoff -a    # turn off ALL swap sources
sudo swapon -a     # read fstab and activate ALL swap entries listed

# Verify
free -h
# Swap: 4.0Gi ✅ confirms fstab entry works

# Final proof — real reboot
sudo reboot
free -h
# Swap: 4.0Gi ✅ survived!
```

---

### ✅ swapoff -a vs swapon -a

| Command | What it does |
|---------|---------------|
| `swapoff -a` | Deactivates ALL swap sources currently active |
| `swapon -a` | Reads `/etc/fstab` and activates ALL swap entries listed there |

Running these two together **simulates a reboot** without actually rebooting — a fast way to test if your fstab entry is correct.

---

## Quick Reference Card

```bash
# Check memory and swap
free -h

# Check available disk space
df -h /

# Create a swap file
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make it permanent
sudo nano /etc/fstab
# /swapfile    none    swap    defaults    0 0

# Test fstab without rebooting
sudo swapoff -a
sudo swapon -a

# Reboot to confirm
sudo reboot
free -h

# Turn off a specific swap file (not all)
sudo swapoff /swapfile

# View active swap devices
swapon --show
cat /proc/swaps
```

---

## Interview Questions

### Basic Level

**Q: What is swap space?**
> Swap is disk space that acts as an overflow for RAM. When physical RAM fills up, Linux moves idle/inactive data to swap to free up RAM and prevent the system from crashing.

**Q: Is swap as fast as RAM?**
> No. Swap lives on disk, which is significantly slower than RAM chips. Swap exists to prevent crashes, not to improve performance. Heavy reliance on swap is a sign a server needs more RAM.

**Q: What are the two ways to create swap space?**
> A swap partition (created during OS install, fixed size) or a swap file (created anytime, flexible, easy to resize). Swap files are more common and flexible in real-world administration.

**Q: What command creates instantly-sized swap file space?**
> `sudo fallocate -l 2G /swapfile` — reserves 2GB instantly at the filesystem level without writing data byte by byte.

**Q: Why do you run chmod 600 on a swap file?**
> Swap can temporarily hold sensitive data copied out of RAM. Setting permissions to 600 ensures only root can read or write to it, preventing other users from accessing potentially sensitive memory contents.

---

### Intermediate Level

**Q: What is the difference between fallocate and dd for creating a swap file?**
> `dd if=/dev/zero of=/swapfile bs=1M count=2048` actually writes zeros to disk byte by byte — slow for large files. `fallocate -l 2G /swapfile` reserves the space instantly at the filesystem level without writing data first.

**Q: What is the full command sequence to create and activate a swap file?**
> `fallocate -l 2G /swapfile` → `chmod 600 /swapfile` → `mkswap /swapfile` → `swapon /swapfile`. Then add an fstab entry to make it permanent.

**Q: What does the fstab entry for a swap file look like, and why is the mount point "none"?**
> `/swapfile none swap defaults 0 0`. The mount point is "none" because swap is not a browsable filesystem/folder — it's raw space used directly by the kernel for memory overflow.

**Q: How do you test if a new fstab swap entry works without rebooting?**
> `sudo swapoff -a` (deactivate all swap) followed by `sudo swapon -a` (reactivate everything listed in fstab). If `free -h` shows the correct total afterward, the fstab entry is correct.

---

### Scenario Based

**Q: A server keeps running out of memory and crashing during traffic spikes. How would swap help, and what are its limits?**
> Adding swap space gives the system breathing room to survive temporary memory spikes without crashing, since it can offload idle data to disk. However, swap is much slower than RAM — if the server is CONSTANTLY relying on swap, that indicates it genuinely needs more physical RAM. Swap is a safety net, not a long-term performance solution.

**Q: You created a swap file, activated it with swapon, but after reboot free -h shows the old (lower) swap total. What went wrong?**
> The swap file was activated temporarily with `swapon` but never added to `/etc/fstab`. Without an fstab entry, swap files (just like disk mounts) do not persist across reboots. Fix: add `/swapfile none swap defaults 0 0` to fstab, then test with `swapoff -a && swapon -a`.

**Q: How would you remove a swap file completely from a server?**
> `sudo swapoff /swapfile` (deactivate it) → remove its line from `/etc/fstab` → `sudo rm /swapfile` (delete the file). Always deactivate before deleting or removing from fstab.

---

*Notes by Pujan | RHCSA Hands-On Self Challenge | Week 2 Part 3*
