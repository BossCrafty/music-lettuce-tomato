# music-lettuce-tomato
documentation on the sequenced music format in Harvest Moon: Magical Melody

---

Harvest Moon: Magical Melody was a GameCube game that used sequenced music (it was later ported to Wii but the filesystem and formats were identical). I couldn't find any info on the internet about the format of the sequenced music, so I'm sticking my own research here. In the future I'd like to make as program that automatically rips MIDIs, converts the soundbanks, and produces a WAV file of all the music, but I'm nowhere near that level yet. In the meantime, here's what I've found.

---

The game's music files are found inside the `sound` folder in the game's filesystem. There's a couple of different types, but the most important ones are the `bgm##.mlt`, ranging from bgm01 to bgm43. Each of these files is a single song, containing all the instrument samples, configuration of the instruments, and the music sequence.

Sequences are literally straight midis, in a hex editor you can open up an mlt file and select the whole area starting with "MThd" (4D 54 68 64) and ending with "ÿ/�" (FF 2F 00), export it to a new file, and slap a .mid to the filename and boom you can play it in any midi player.
