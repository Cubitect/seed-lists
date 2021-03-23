More than one?

We have found that we can move quad-huts around by transforming the seed, but
can we have multiple quad-huts in the same world? Well, given that we have an
exhausted list of all seeds that have a quad-hut at the origin (which I will
call seed-bases), in order for a seed to contain a second quad-hut, there must
exist a `moveStructure()` operation on one of these seed bases that gives us a
second seed that is also in the list. Unfortunately, checking this is not quite
that straight forwards, since there are at worst round 750k seeds in the list
and a minecraft world has around `(60M / 512)^2 ~ 1.37x10^10` potential move
transformations, and each result has to be checked against all the other seeds
in the list. However, we can again exploit that there are only a handful of
20-bit values that the seed bases can end with (generated from `low20Quad*`).
As a result we only need to examine move transformations that satisfy the
modular equation:
```
A*x + B*z + seed = Q | mod M
```
where `M = 2^20`, `A = 341873128712` and `B = 132897987541` which come from
`moveStructure`. The `seed` is a base from our list, and `Q` is a number
from `low20Quad*`. This equation can be solved for `z`, given an `x`, by finding
the modular inverse of `B | mod M` (note that we cannot solve for `x`, because
`A` and `M` are both divisible by two, which means they are not co-prime and `A`
has no modular inverse). So we get:
```
z = (Q - A*x - seed) * inv(B) | mod M
```
So if we choose `x`, we can calculate a value for `z` and we only need to check
other values for `z` that are 2^20 apart. This reduces the search space enough
to put it into the realm of things that can be checked within a reasonable
amount of time. This type of search can be done with `scanForQuads`. This
function generates (x,z) pairs using this formula, applies the move
transformation to an input seed, and then checks if the seed would be a quad-hut
(which is faster than to compare against the list of 750k bases). A full
program that checks for quad-hut potential within half a million blocks might
look something like this:


```C
#include "cubiomes/finders.h"

#include <stdio.h>

// tests if (s48-salt) is a quad-hut base
int check(int64_t s48, void *data)
{
    const StructureConfig sconf = *(const StructureConfig*) data;
    return isQuadBase(sconf, s48 - sconf.salt, 128);
}

int main()
{
    initBiomes();

    StructureConfig sconf = SWAMP_HUT_CONFIG;
    const int64_t *lbits = low20QuadHutNormal;
    int lbitcnt = sizeof(low20QuadHutNormal) / sizeof(int64_t);
    int64_t *protoBases = NULL;
    int64_t protoBaseCnt = 0;

    printf("find bases...\n");
    int err = searchAll48(
            &protoBases, &protoBaseCnt, NULL, 8,
            lbits, lbitcnt, 20, check, &sconf);
    if (err)
        exit(1);

    printf("scaning... \n");
    int64_t maxreg = 512000 / 512;
    int x = -maxreg, z = -maxreg, w = 2*maxreg, h = 2*maxreg;
    int dumplist = 1;

    int64_t pbi;
    for (pbi = 0; pbi < protoBaseCnt; pbi++)
    {
        Pos qh[100];
        int64_t protobase = protoBases[pbi];
        int64_t base = protobase - sconf.salt;
        int i, j, k, qhcnt;
        qhcnt = scanForQuads(sconf, 128, base, lbits, lbitcnt, 20, sconf.salt, 
                x, z, w, h, qh, 100);

        if (qhcnt < 2)
            continue;

        printf("%15ld %5lx : ", base, protobase & 0xfffff);
        if (dumplist) // print region coordinates
            for (j = 0; j < qhcnt; j++)
                printf(" [%5d,%5d]", qh[j].x, qh[j].z);
        else
            printf("# = %d", qhcnt);
        printf("\n");
    }

    return 0;
}
```

Output:

```
find bases...
scaning... 
   517358846628 89718 :  [ -312, -576] [    0,    0]
  1170042880676 89718 :  [  -76, -246] [    0,    0]
  1376352305828 89718 :  [ -312, -576] [    0,    0]
  2862864738980 43f18 :  [    0,    0] [ -141, -526]
  2991006253732 89718 :  [  -76, -246] [    0,    0]
  3223170417316 89718 :  [  -76, -246] [    0,    0]
  4812108038820 89718 :  [  -76, -246] [    0,    0]
  5733580816804 75618 :  [    0,    0] [  312,  576]
  6592574276004 75618 :  [    0,    0] [  312,  576]
  6865235575460 89718 :  [  -76, -246] [    0,    0]
  8903664108950 79a0a :  [    0,    0] [   76,  246]
  8964866344342 79a0a :  [  141,  526] [    0,    0]
 10114711717270 79a0a :  [    0,    0] [   76,  246]
 10139159331492 89718 :  [ -312, -576] [    0,    0]
 11694116734358 79a0a :  [    0,    0] [   76,  246]
 12243106867876 43f18 :  [    0,    0] [ -141, -526]
...
277586757042596 75618 :  [    0,    0] [  312,  576]
277998000243364 89718 :  [  -76, -246] [    0,    0]
278445750501796 75618 :  [    0,    0] [  312,  576]
278945271154070 79a0a :  [    0,    0] [   76,  246]
279418993673622 79a0a :  [    0,    0] [   76,  246]
279818963616420 89718 :  [  -76, -246] [    0,    0]
280342814360228 89718 :  [  -76, -246] [    0,    0]
280998398690710 79a0a :  [    0,    0] [   76,  246]
```

There are few things to notice here. First, one of the region coordinate pairs
is always the origin, which just comes from the fact that we are looking at
quad-hut bases. However, there is a second quad-hut attempt, which seems to be
placed at one of only a hand full of distinct positions. The restrictions on
this position correspond to a relative offset between quad-huts. This becomes
even more apparent when we recognise that for each occuring set of coordinates,
the offset in the other direction is also valid, e.g: for `[-312,-576];[0,0]`
there is a corresponding `[0,0];[312,567]`. This scarcity of distinct offsets
is bad news if we want to find seeds which have two or more quad-huts in
relative proximity to one another (In the results so far, the closest any
quad-huts can be together is a distance of over 130k blocks appart!). However,
if we relax our requirements for quad-huts and include the `low20QuadHutBarely`
constellations then we find that there is one offset which has two quad-huts
that are remarkably close together:

```
  5065407553702 c751a :  [    0,    0] [  -52,   34]
  7851460513104 ee1c4 :  [   52,  -34] [    0,    0]
 12972202380624 ee1c4 :  [   52,  -34] [    0,    0]
 17473260997968 ee1c4 :  [   52,  -34] [    0,    0]
 21110331629734 c751a :  [    0,    0] [  -52,   34]
 26231073497254 c751a :  [    0,    0] [  -52,   34]
...
260873659703462 c751a :  [    0,    0] [  -52,   34]
263659712662864 ee1c4 :  [   52,  -34] [    0,    0]
273281513147728 ee1c4 :  [   52,  -34] [    0,    0]
276918583779494 c751a :  [    0,    0] [  -52,   34]
```

This means we can find a whole family of seeds with two quad-huts less than 
32k blocks apart! Of these, the North-East quad-hut has an ideal constellation,
and the other is just barely a quad-hut. Building upon this, we are also free
to place one of those quad-huts near the origin and move the other either to
`(-52*512, -34*512)` or to `(52*512,34*512)`, or we can have both approximately
equidistant to the origin. This would put them near `(-13312,-8704)` and
`(13312,8704)`, only 16k blocks from spawn! Of course there is one final issue.
We require a swamp biome at each of those eight swamp huts. This is a major
restriction, even with the freedom to choose the top 16-bits.

Below is a full program that finds all double quad-huts for which all eight
swamp huts are less than 20000 blocks (radially) from the origin. Running this
code till completion takes some time, but the output can be found
[here for 1.12](https://github.com/Cubitect/seed-lists/dqh20k_1_12.txt) and 
[here for 1.16](https://github.com/Cubitect/seed-lists/dqh20k_1_16.txt).

```C
#include "cubiomes/finders.h"

#include <stdio.h>

// tests if (s48-salt) is a quad-hut base
int check(int64_t s48, void *data)
{
    const StructureConfig sconf = *(const StructureConfig*) data;
    return isQuadBase(sconf, s48 - sconf.salt, 128);
}

int main()
{
    initBiomes();

    StructureConfig sconf = SWAMP_HUT_CONFIG;
    const int64_t *lbits = low20QuadHutBarely;
    int lbitcnt = sizeof(low20QuadHutBarely) / sizeof(int64_t);
    int64_t *protoBases = NULL;
    int64_t protoBaseCnt = 0;

    printf("find bases...\n");
    int err = searchAll48(
        &protoBases, &protoBaseCnt, NULL, 8,
        lbits, lbitcnt, 20, check, &sconf);
    if (err)
        exit(1);

    printf("scaning... \n");
    int64_t maxrad = 20000;
    int64_t maxreg = maxrad / 512;
    int x = -2*maxreg, z = -2*maxreg, w = 4*maxreg, h = 4*maxreg;
    int mc = MC_1_16;
    LayerStack g;
    setupGenerator(&g, mc);

    int64_t pbi;
    for (pbi = 0; pbi < protoBaseCnt; pbi++)
    {
        Pos qh[100];
        int64_t base = protoBases[pbi]-sconf.salt;
        int i, j, k, qhcnt;
        qhcnt = scanForQuads(
                sconf, 128, base, lbits, lbitcnt, 20, 
                sconf.salt, x, z, w, h, qh, 100);

        if (qhcnt >= 100)
            printf("Buffer insufficient!\n");
        if (qhcnt < 2)
            continue;

        if (qh[0].x != 0 || qh[0].z != 0)
            continue; // avoid duplicates by forcing the origin quad-hut

        // transpose the protobase for all offsets that are possibly in range
        for (i = -maxreg; i < maxreg; i++)
        {
            for (j = -maxreg; j < maxreg; j++)
            {
                int64_t s48 = moveStructure(base, i, j);
                // get the position of all 8 swamp huts
                Pos p[8];
                int styp = sconf.structType;
                getStructurePos(styp, mc, s48, qh[0].x+i+0, qh[0].z+j+0, p+0);
                getStructurePos(styp, mc, s48, qh[0].x+i+0, qh[0].z+j+1, p+1);
                getStructurePos(styp, mc, s48, qh[0].x+i+1, qh[0].z+j+0, p+2);
                getStructurePos(styp, mc, s48, qh[0].x+i+1, qh[0].z+j+1, p+3);
                getStructurePos(styp, mc, s48, qh[1].x+i+0, qh[1].z+j+0, p+4);
                getStructurePos(styp, mc, s48, qh[1].x+i+0, qh[1].z+j+1, p+5);
                getStructurePos(styp, mc, s48, qh[1].x+i+1, qh[1].z+j+0, p+6);
                getStructurePos(styp, mc, s48, qh[1].x+i+1, qh[1].z+j+1, p+7);

                // check that all swamp-huts are within 'maxrad' of the origin
                for (k = 0; k < 8; k++)
                {
                    int64_t dx = p[k].x, dz = p[k].z;
                    if (dx*dx + dz*dz > maxrad*maxrad)
                        goto L_next_pos;
                }

                // look for valid 64-bits seeds
                int64_t high;
                for (high = 0; high < 0x10000; high++)
                {
                    int64_t seed = s48 | (high << 48);
                    for (k = 0; k < 8; k++)
                    {
                        if (!isViableStructurePos(
                            sconf.structType, mc, &g, seed, p[k].x, p[k].z))
                        {
                            goto L_next_high;
                        }
                    }

                    // we found a seed with two quad-huts!
                    printf("%20ld @ [%6d,%6d] [%6d,%6d]\n",
                            seed, p[0].x, p[0].z, p[4].x, p[4].z);

                    L_next_high:;
                }
                L_next_pos:;
            }
        }
    }

    return 0;
}
```

Another thing we could do is look for seeds that have as many quad-huts as
possible. To explore this, we can start by adapting the code from earlier, but
increase the search radius from 1000 to 30e6/512 ~ 58594 regions. By doing this
we find that there are 48-bit seeds that have as many as 14 ideal quad-hut
generation attempts. Indeed if we examine random seeds, we find that most seeds
have at least some quad-hut potential. So the problem shifts to finding the
necessary biomes rather than suitable structure positions. This also means that
an exhaustive search becomes impractical and it makes more sense to impose some
restrictions on the selection of seeds. For instance we can take into account
the distance to the nearest quad-hut or limit the range for the furthest
quad-huts.

After some tinkering and a lot of messy code later I have found some seeds that
have four and even five quad-huts, mostly based on ideal constellations. Some 
of these can be found 
[here](https://htmlpreview.github.com/?https://github.com/Cubitect/seed-lists/mqh_1_16.html).
There almost certainly exist seeds with six or more quad-huts, but they are
rather difficult to find without further optimizations.







