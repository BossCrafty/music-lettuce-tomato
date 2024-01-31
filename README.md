# music-lettuce-tomato
documentation on the sequenced music format in Harvest Moon: Magical Melody

---

Harvest Moon: Magical Melody was a GameCube game that used sequenced music (it was later ported to Wii but the filesystem and formats were identical). I couldn't find any info on the internet about the format of the sequenced music, so I'm sticking my own research here. In the future I'd like to make as program that automatically rips MIDIs, converts the soundbanks, and produces a clean WAV file of the music, but I'm nowhere near that level yet. In the meantime, here's what I've found.

---

The game's music files are found inside the `sound` folder in the game's filesystem. There's a couple of different types, but the most important ones are the `bgm##.mlt`, ranging from bgm01 to bgm43. Each of these files is a single song, containing all the instrument samples, configuration of the instruments, and the music sequence.

Sequences are literally straight midis, in a hex editor you can open up an mlt file and select the whole area starting with "MThd" (4D 54 68 64) and ending with "ÿ/�" (FF 2F 00), export it to a new file, and slap a .mid to the filename and boom you can play it in any midi player.

---
# Detailed Findings:

## Some things to take note of:
-	All pointers and length counts and other numbers in these files are in Big Endian
-	If the last byte of a chunk doesn't fall on an F (aka the last column in a hex editor), then it will be padded out with 0x55 / "U" so that the header of the chunk after it is on a round number.
	So for example, if the last byte of a chunk is on $6CC13, then there will be 12 0x55s so that the next header is precisely at $6CC20
	HOWEVER these filler bytes do not count towards the lengths stated in the headers.


### First is the main header of the entire file. consists of 16 bytes:
- 8 bytes for the string "`gcaxMLT `"
- 4 bytes of zeros
- 4 bytes representing the size of the file, in big endian order (which includes this header)


### immediately after, at $10, is the MLTM chunk. I have not figured out what this chunk means.
The header is:
- 8 bytes for the string "`gcaxMLTM`"
- 4 bytes of zeros
- 4 bytes representing the length of the MLTM chunk (which does **NOT** include this header)

### after the end of this is the MPB Chunk, which contains both samples and the instrument config.
The header is:
- 8 bytes for the string "`gcaxMPB `"
- 4 bytes of zeros
- 4 bytes representing the length of the MPB chunk (which does NOT include this header) 

### Next is the MPBW chunk, which holds all the audio samples.
You ought to be familiar with the header format by now:
- 8 bytes for the strings "`gcaxMPBW`"
- 4 bytes of zeros
- 4 bytes for the length (which does NOT include the header

Then there's the big ol block of samples. The start of each sample is specified in the sample config, which is part of the MPBP chunk below. <br/>
Most of the samples are raw	PCM audio, and you can import the whole thing audacity as Raw audio with these settings: <br/>
  `16-bit WAV, Big-endian, mono, and 32,000 hz` <br/>
However, there are sometimes samples that compressed, encoded with DSP-ADPCM. These will just be noise in audacity, and will require some additional info to decode which is also found in the sample config below. <br/>


### Then there's the MPBP chunk, which configures the instruments
Classic header:
- 8 bytes for the strings "`gcaxMPBP`"
- 4 bytes of zeros
- 4 bytes for the length (which does NOT include the header)
	
Now, this chunk is split into two parts, which I call the **Sample Directory table** and the **Instrument Definition table**.
The first thing in the chunk is 0x20 bytes of mostly unknown purpose, however I'm pretty sure the first byte is the number of samples

Next is the Sample Directory table. Each entry is 0x50 bytes.
The layout of this data is shown in the image I made, but to summarize:
- $08 - sample rate (4 bytes)
- $0D - a single byte which indicates whether the sample loops or not
- $10 - loop start (4 bytes)
- $14 - loop end (4 bytes)
- $4A - a single byte seems to always be 0A in raw samples, and 00 for ADPCM samples.
- $4C - Sample offset pointer (4 bytes), starting from beginning of sample data (immediately after 0x60 byte headers. so if the pointer is zero, the sample starts at $60, if the pointer is B160, then the sample start at B1C0)

(INSERT IMAGE HERE)

Despite being a whole 0x50 bytes, nearly everything that isn't listed here is usually set to zero— Except if the sample is ADPCM. <br/>
When working with the Wii and GameCube, DSP-ADPCM samples typically include a header containing info necessary for them to be decoded. In Harvest Moon, these samples were stripped of their header and shoved in with the rest of the raw samples... <br/>
And that header actually becomes their entry in the Sample Directory table! Meaning, for ADPCM samples, you can simply copy their 0x50 byte table entry, stick it to the beginning of the sample, and throw it into a decoder.

Interestingly, some of the info like the loop points still match up in these entries whether it's compressed or not. <br/>
My hypothesis is that the entry format was based on the DSP-ADPCM header, then simply stripped of the stuff that's not needed for raw samples. <br/>

Anyway, after the last sample directory entry is a weird 0x80 byte block that just counts from 00 to 7F

### Next up is the Instrument Definition table! <br/>
First off is a series of pointers to each entry (because the entries can vary in size) <br/>
Each pointer is 4 bytes, however the first byte of every single one is always 01 , so really treat the latter 3 bytes as the pointer. <br/>
These pointers are offsets from the beginnings of the MPBP chunk, with the first byte after the header being $0. <br/>
So for example, if the "gcaxMPBP" header is at $6BF20, then the beginning of the chunk is at $6BF30, and thus a pointer of 07 14 means that instrument starts at $6C644. <br/>

As for the Instrument Definitions themselves, each is a MINIMUM of 0x50 bytes. <br/>
it consists of a 0x10 byte header, followed by one or more 0x40 byte blocks. <br/>
The header is as follows:
- $00 - a single byte stating how many samples are in this instrument. 
- $02 - two byte pointer that seems to just point to the beginning of the instrument data immediately after the header? headers are consistently 10 bytes so I don't get why this pointer seems to exist to tell where the header ends???
- $04 - two repeated bytes of unknown purpose. The only two options I've seen are "02 02" and "0C 0C". 02 seems more common but I can't find any pattern to what they mean.
- $06 - 10 zeros filling the rest of the header.

Then follow a series of 0x40 byte blocks, how many are determined by how many samples are in the instrument. <br/>
So if an instrument only has a single sample, the whole entry will be 0x50 bytes. If an instrument has two samples, it will be 0x90 bytes, and so on. An 18 sample instrument will be 0x490 bytes!

As for the blocks themselves, again each is 0x40 bytes. <br/>
I haven't finished figuring out the configuration, but here's what I have so far:
- $03 - a single byte that determines which sample to use
- $0C - this is the root key I think
- Ugh I know this has to contain all sorts of other stuff like key ranges and ADSR but I can't figure it out
	
ok anyway, that's finally the end of the huge MPB chunk!
	
### After all that, next is the MSB chunk. This seems to contain solely another chunk, which holds the Midi.
good ol header format:
- 8 bytes for the strings "`gcaxMSB `"
- 4 bytes of zeros
- 4 bytes for the length (which does NOT include the header)

Immediately after are 0x10 bytes of unknown purpose. They're all zeros except the 4th one which is 10 (haven't confirmed this is consistent)

### Next is the MSBM chunk, this is the chunk where the Midi is.
This header appears to be 0x12 bytes for some reason:
- 8 bytes for the strings "`gcaxMSM`"
- 4 bytes of zeros
- 4 bytes for the length of the Midi (which does NOT include the header)
- 2 MORE zero bytes!! (why, why is this header difference, what purpose do these bytes serve)
	
After those 2 zero bytes that exist for some reason, the midi begins, starting with the string "MThd" and ending with "�ÿ/�" aka bytes 00 FF 2F 00. <br/>
This midi is a completely standard midi, you can select this whole block and export it as an individual file, slap a .mid file extension on it, and it'll play in any midi player. <br/>
However, I have found that things like the pitch bend range being non-standard, so pitch bends will be wayy smaller in standard midi players than they should be. <br/>

Then there might be a few 0x55 bytes padding it out to end on the 0f column, and then there might be a single row of 0x10 zero bytes, and that's the end of the MLT file!
