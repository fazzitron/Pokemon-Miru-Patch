# Pokemon-Miru-Patch
Patch to remove seed generation from the Gen 3 Kanto Games

## Reasoning
Pokemon Fire Red & Leaf Green have notoriously difficult RNG Manipulation. That's because it starts its seed generation using a very fast timer

```
void SeedRngAndSetTrainerId(void)
{
    u16 val = REG_TM1CNT_L; // <-- The timer used to generate the seed, very fast!
    SeedRng(val);           // <-- The value where the seed is stored
    REG_TM1CNT_H = 0;
    gTrainerId = val;       // <-- Also based on the seed
}
```

## Method
Unfortunately, it's not as easy as setting val to 0x00 as that seems to mess with trainer ID generation. Not only that, but it's tricky to do reliably with the amount of space we have to work with. So, the easiest way to set the seed is to just make sure its never set to begin with.

To do this without breaking things, we need to alter this block of code in title_screen.c:

```
    case 2:
        if (!gPaletteFade.active)
        {
            SeedRngAndSetTrainerId(); //<--This is the line that makes RNG manipulation so difficult!
            SetSaveBlocksPointers();
            ResetMenuAndMonGlobals();
            Save_ResetSaveCounters();
            LoadGameSave(SAVE_NORMAL);
            if (gSaveFileStatus == SAVE_STATUS_EMPTY || gSaveFileStatus == SAVE_STATUS_INVALID)
                Sav2_ClearSetDefault();
            SetPokemonCryStereo(gSaveBlock2Ptr->optionsSound);
            InitHeap(gHeap, HEAP_SIZE);
            SetMainCallback2(CB2_InitMainMenu);
            DestroyTask(FindTaskIdByFunc(Task_TitleScreenMain));
        }
        break;
```

We need to remove the function call SeedRngAndSetTrainerId(). But, we can't just remove it by itself without mangling addresses needed for other ROM hacks. So, we have to go to the assembly code to remove it safely.

Using a decompiler like Ghidra or by compiling the repository at https://github.com/pret/pokefirered, we can find the following compilation of the method above (functions renamed for ease of reading):

```
        08079252 28 d1           bne        gPaletteFade_active_notZero
        08079254 87 f7 86 f9     bl         SeedRngAndSetTrainerId  ; <-- The bytes that need removed
        08079258 d2 f7 fe fe     bl         SetSaveBlocksPointers
        0807925c db f7 e4 fb     bl         ResetMenuAndMonGlobals
        08079260 60 f0 76 fa     bl         Save_ResetSaveCounters
        08079264 00 20           movs       r0,#0x0
        08079266 61 f0 49 f9     bl         LoadGameSave
```
We need to replace these bytes with NOP calls. But, we have to be careful about being in either THUMB or ARM mode. In this case, we'll be in THUMB mode. So, we'll replace it with 2 commands instead of 1:

```
        08079252 28 d1           bne        gPaletteFade_active_notZero
        08079254 00 1c           adds       r0,r0,#0x0            ; <-- Our patched bytes
        08079256 00 1c           adds       r0,r0,#0x0            ;     right here!
        08079258 d2 f7 fe fe     bl         SetSaveBlocksPointers
        0807925c db f7 e4 fb     bl         ResetMenuAndMonGlobals
        08079260 60 f0 76 fa     bl         Save_ResetSaveCounters
        08079264 00 20           movs       r0,#0x0
        08079266 61 f0 49 f9     bl         LoadGameSave
```

Now our patch works! It does mean that REG_TM1CNT timer never gets stopped. But, that's hard to fix without either inserting custom code or breaking the Trainer ID.

And just like that, our patch is done! Unless...

## Future: Thumim
More on this soon...hopefully. Assuming I ever get it to work...

## Title Screen
Since this is a modification that could be used to try to gain an advantage in things such as speedruns, the Title Screen has been modified in an attempt to thwart such things. It's not the best looking, but it's enough to be seen.

## Tools Used
* [Pokemon Fire Red Decompilation](https://github.com/pret/pokefirered)
* [pokemonrng.com](https://pokemonrng.com)
* WSL
* mGBA
* GDB
* Ghidra
