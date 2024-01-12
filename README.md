# Zápočtový program

Úloha je vytvořit hru napodobující původního Pac-Mana. Hru jsem vyráběl v Unity a snažil jsem se zachoval všechny herní mechaniky, pixel art a zvuky z původní hry. Kód je rozdělený do několika C# skriptů, které spolu prostřednictvím Unity komunikují. Nemusel jsem řešit žádné algoritmicky nebo výpočetně náročné úlohy. Hlavní problém byl správně navrhnout, který skript má dělat co a jak spolu mají jednotlivé skripty komunikovat. 

## Uživatelská část dokumentace

### Co je to Pac-Man?

Pacman je hra z osmdesátých let, ve které je hráč spolu se čtyřmi duchy umístěn do bludiště plného teček. Hráč ovládá postavičku se jménem Pac-Man. Cílem je sníst všechny tečky a postoupit do další úrovně. Duchové - Blinky (červený), Pinky (růžový), Inky (modrý) a Clyde (oranžový) - se mezitím snaží hráče chytit. Pokud Pacman sní velkou tečku, tak se duchové vyděsí a na chvíli se stanou zranitelní. Zranitelné duchy je možné sníst a získat tak bonusové body. Pokud Pacman sní všechny čtyři duchy, tak získá život navíc. Také se v každé úrovni dvakrát objeví bonusový symbol (většinou se mu říká ovoce), který je možné sníst pro body navíc. Potom, co hráč sní všech 244 teček, tak postoupí do další úrovně, kde jsou duchové rychlejší, agresivnější a zranitelní po kratší dobu. Hra končí když hráč ztratí poslední život.

### Ovládání

Hráč udává Pacmanův směr pomocí šipek nebo WSAD. Je možné dělat takzvaný pre-turn nebo post-turn, tedy zahájit akci zatáčení pár pixelů před (nebo za) středem odbočky. Díky tomu může hráč zatáčet rychleji než duchové.

Hru je možné pozastavit pomocí `Escape` nebo `Space`. Na pause screen je tlačítko na ovládání hlasitosti hudby a tlačítko pro návrat na úvodní obrazovku. 

## Programátorská část dokumentace

Snažil jsem se najít nějaký zdroj informací o původní hře, ze kterého bych mohl vycházet. Našel jsem [The Pac-Man Dossier](https://pacman.holenet.info/), což je velmi podrobný popis všech mechanik této hry. Včetně tabulek udávajících různé parametry v jednotlivých úrovních. Snažil jsem se všechno implementovat podle této specifikace. Jediná věc, kterou jsem udělal jinak, je detekce kolizí, na kterou jsem použil vestavěný kolizní systém Unity. V této části dokumentace se pokusím stručně popsat ty nejdůležitější specifikace hry. Nebudu zabíhat do zbytečných detailů, ty se případně dají dohledat v již zmíněném [The Pac-Man Dossier](https://pacman.holenet.info/). Tato část bude psaná anglicky, protože mi to přijde přirozenější. 

### Game Levels 

There are no major changes to the game as the player progresses through the levels, just some value tweaks. The important thing to note is that the last change is in level 21 - all subsequent levels are the same.

### Ghost Modes

Ghosts have 4 different modes of behaviour they can be in, which we represent with an enum:

* Chase - chase after Pacman, every ghosts targets a different tile
* Scatter - run back into one of the four corners of the maze, every ghost has their favourite one
* Frightened (fright) - move randomly through the maze. Ghosts in this mode can be eaten.
* Dead - return back to the ghost house. Dead ghosts cannot harm Pacman.

Ghosts cannot directly reverse their direction, but the system can sometimes force them to reverse direction. This happens when they change between the scatter and chase modes and when a power dot is eaten.

### Scatter-Chase Transitions

Ghosts start in the scatter mode and switch to chase mode after a few seconds. Then after chasing Pacman for a while, they scatter again and the whole thing repeats. Ghosts scatter like this only four times, then begins an indefinite chase period. This behavior resets every time Pacman loses a life and at the beginning of a new level. 

### Frightened Behavior

Ghosts enter fright mode and reverse their direction whenever Pacman eats a power dot. In later levels, they don't get frightened at all and only reverse their direction. Frightened ghosts move randomly through the maze. The random number generator gets reset with the same seed at the start of each new level and whenever a life is lost. This results in frightened ghosts always choosing the same paths when executing patterns during play. It is possible to gain bonus lives by eating all four ghosts during a single fright period.

### Speed

The game starts with Pacman moving at 80% of his maximum speed. He gets faster as the game progresses but his speed gets reduced to 90% in level 21. On the other hand, ghosts get faster. Blinky is special since he transforms into "Cruise Elroy" after a certain number of dots has been eaten. Elroy is faster than the other ghosts. Ghosts slow down when they get frightened, while Pacman speeds up.

### Special Areas

There are 2 types of special zones in the game

* the tunnel - it teleports the ghosts and Pacman to the other side. Ghosts are slowed down in the tunnel.
* the red zone - ghosts can't move vertically there. This restriction doesn't apply to dead ghosts. There is a red zone directly above the ghost house and a second one above the lowest T shaped obstacle.

### Ghost House Logic

Blinky starts outside of the house, other ghosts start inside. Ghosts return to the house after being eaten. The leaving priority is as follows: Blinky > Pinky > Inky > Clyde. Blinky always leaves immediately. The other ghosts use dot counters and a timer to determine who should leave next. A special global counter is used after Pacman dies. I recommend reading this [section](https://pacman.holenet.info/#CH2_Home_Sweet_Home) of the Pac-Man Dossier.

### Ghost Movement

The maze is divied into 8x8 pixel tiles. Whenever a ghost is in chase or scatter mode, they are trying to reach a **target tile**. For scatter mode its one of the corners of the maze. Every ghost targets a different tile while in chase mode, but it is always somehow liked to Pacman.

* Blinky - directly targets Pacman

* Pinky - targets the tile 4 tiles ahead of Pacman. But if Pacman is moving up, then Pinky targets the tile four tiles up and four tiles to the left. This was caused by an overflow bug in the original game.

* Inky - uses a special middle tile. The middle tile is the tile 2 tiles ahead of Pacman. Now draw a vector from Blinky to the middle tile, then add this vector to the middle tile and you will get Inkys target tile.

* Clyde - targets Pacman if the distance between them is at least 8 tiles. If he gets too close to Pacman, he runs back to his corner.

Ghosts also use a target tile to return home when they die.

### Picking The Next Tile

Ghosts always plan one step into the future. 
If a ghost just entered tile A, then he already knows the next tile he will enter - let's call it B. 
After entering tile A, the ghost chooses a direction he will set once he reaches the center of tile B.
After reaching the center of tile A, he will set the direction he calculated on the previous tile

Ghosts pick their next tile only based on the euclidean distance between the next tile and the target tile.
If a ghost has to choose between more tiles that are the same distance from the target tile, then he prefers directions in this order: up, left, down, right. 

### Fruit 

Fruit appears below the ghost house after the first 70 and the first 170 dots have been eaten. It remains there for a random amount of time between 9 and 10 seconds. Pacman can eat the fruit for bonus points.

### Representation Of The Maze

The maze layout is stored in a plain text file and is loaded into a 2D array at the start of the game. The array can be queried for information about walls, position of the dots, red zones etc.


## Credits 

I used the following online assets when creating the game

* [maze and sprites](https://www.spriters-resource.com/arcade/pacman/sheet/52631/)
* [start screen and some UI elements](https://www.spriters-resource.com/arcade/pacman/sheet/113279/)
* [font](https://www.spriters-resource.com/arcade/pacman/sheet/73388/)
* [sound effects](https://www.sounds-resource.com/arcade/pacman/sound/10603/)

