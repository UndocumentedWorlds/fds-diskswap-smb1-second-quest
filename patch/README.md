# FDS support for 100thCoin's BizHawk hot-swap fork

100thCoin's fork (https://github.com/100thCoin/BizHawk) implements
`HotSwap()` for NES cartridges, but it fails on `.fds` images because
the FDS BIOS is never passed to `Init()`.

In `NES.Core.cs`, `Init()` throws immediately:

```csharp
if (fdsbios == null)
    throw new MissingFirmwareException("Missing FDS Bios");
```

while `HotSwap()` calls `Init(null, rom, null)`.

The fix is to keep the BIOS obtained during the normal load and pass it
through.

## Changes

### src/BizHawk.Emulation.Cores/Consoles/Nintendo/NES/NES.Core.cs

Add a field:

```csharp
public byte[] hotSwapFdsBios;
```

Replace `HotSwap()`:

```csharp
public void HotSwap(string filePath)
{
    var file = new HawkFile(filePath, false, false);
    if (file.Exists)
    {
        byte[] rom = file.ReadAllBytes();

        byte[] fdsbios = null;
        if (rom.Length >= 4
            && (rom.Take(4).SequenceEqual(System.Text.Encoding.ASCII.GetBytes("FDS\x1A"))
                || rom.Take(4).SequenceEqual(System.Text.Encoding.ASCII.GetBytes("\x01*NI"))))
        {
            fdsbios = hotSwapFdsBios;
        }

        hotSwapping = true;
        Init(null, rom, fdsbios);
        hotSwapping = false;
    }
    else
    {
        hotSwapping = false;
    }
}
```

The header check mirrors the one inside `Init()`, so both headered
`.fds` files and raw disk images (starting with `\x01*NI`) are handled.

### src/BizHawk.Emulation.Cores/Consoles/Nintendo/NES/NES.cs

In the constructor, after the bad-dump correction:

```csharp
var fdsBios = comm.CoreFileProvider.GetFirmware(new("NES", "Bios_FDS"));

if (fdsBios != null && fdsBios.Length == 40976)
{
    // ... existing bad dump handling ...
    fdsBios = tmp;
}

hotSwapFdsBios = fdsBios;      // <-- add this line
```

Placing it after the correction means the swap uses exactly the same
byte array that the normal load path uses.

## Notes

- Internal RAM (`$0000-$07FF`) is preserved by the existing
  `hotSwapping` flag. This is what makes the swap useful.
- The FDS board is recreated by `Init()`, so the 32 KB PRG-RAM is not
  preserved — but it is reloaded from the new disk anyway.
- Building: the solution targets `net48`. `global.json` uses
  `rollForward: latestMajor`, so a modern .NET SDK works. EmuHawk
  requires the Visual C++ 2010 SP1 x64 runtime
  (https://github.com/TASEmulators/BizHawk-Prereqs/releases/latest).

## Usage

Put the `.fds` inside the BizHawk folder. In TAStudio, enter the path
in the "Hot Swap File Path" column and tick **Reset** on the same
frame. `HotSwap()` only fires on a frame where Reset is pressed.