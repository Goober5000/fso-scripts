# SaveLoadX

A streamlined checkpoint and savestate system.

## Notes

- Support ships are never saved.  A ship is treated as a support ship when its class has the `Support` ship type (objecttypes.tbl); the `Support` name prefix is only used as a fallback for ships whose handles can no longer be resolved at save time.  The save log names which route made the decision.
- Loading is tolerant of saves that no longer match the mission.  A ship class, weapon class, or team that no longer exists, or a subsystem or weapon-bank count that has changed, is logged in `fs2_open.log` and skipped; the rest of the ship's data is still applied.  A save file that fails to parse is treated as empty.
- Save slots are stored under string keys in the JSON file.  Files written by earlier versions (array-shaped) are read transparently and rewritten in the new shape on the next save.
- Mission names passed to `lua-savestate-load-external`, `lua-savestate-load-external-var`, and `lua-savestate-check` may be given with or without the `.fs2` extension and are matched case-insensitively.
- Saved subsystem entries carry the subsystem's canonical model name and are restored by name, either at load (ships already present) or on arrival (ships that had not yet arrived), since a parse object only lists the subsystems FRED explicitly wrote.  Saves written before names were recorded are matched by position and say so in the log.
- `lua-savestate-shipstatus` returns true for a ship that has no entry in the save state, matching its documented behaviour of returning true when a ship "is present or hasn't arrived".  A ship that had not arrived when the checkpoint was taken has no entry, so one whose arrival cue uses this SEXP will now arrive normally after a load; previously it never arrived at all.  If you added extra conditions to an arrival cue to work around that, they can come out.
- A ship that was not in the mission when the checkpoint was taken is left entirely alone when it arrives, so the arrival state FRED set up for it is preserved.
