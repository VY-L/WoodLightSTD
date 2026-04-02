Note, when i refer to MC code terminology, i am going to default to yarn mapping
# The Goal
What is a good wood light standardization?

I think the two main standards:

- It should actually standardize: players who do the same thing (up to a point that it is impossible to argue that one play is objectively better than the other) should get the same result
- It should not change any strategies

And also possibly performance may be a third

The second thing is the main problem that makes me not like others' solutions. A lot of times, it is impossible to fully perserve 2 while making any progress in standardization, but i

# Vanilla Mechanisms
There are two events that woodlight uses in vanilla minecraft: Random ticks and fire ticks (which many confuse). Random ticks are what makes lava light things on fire, and fire ticks (which are implemented by scheduled ticks) are what makes fire finish burning stuff and spread

## Random Ticks & Lava
The ingame server picks 3 (or whatever you change gamerule randomTickSpeed to) blocks from every subchunk (16\*16\*16 cube) every tick. This is what governs stuff like crop growing. In this case, when it chooses a lava block, it runs the following code to potentially spawn fire.
```java

    public void onRandomTick(World world, BlockPos blockPos, FluidState fluidState, Random random) {
        if (world.getGameRules().getBoolean(GameRules.DO_FIRE_TICK)) {
            int i = random.nextInt(3);
            if (i > 0) {
                BlockPos blockPos2 = blockPos;

                for (int j = 0; j < i; j++) {
                    blockPos2 = blockPos2.add(random.nextInt(3) - 1, 1, random.nextInt(3) - 1);
                    if (!world.canSetBlock(blockPos2)) {
                        return;
                    }

                    BlockState blockState = world.getBlockState(blockPos2);
                    if (blockState.isAir()) {
                        if (this.canLightFire(world, blockPos2)) {
                            world.setBlockState(blockPos2, AbstractFireBlock.getState(world, blockPos2));
                            return;
                        }
                    } else if (blockState.getMaterial().blocksMovement()) {
                        return;
                    }
                }
            } else {
                for (int k = 0; k < 3; k++) {
                    BlockPos blockPos3 = blockPos.add(random.nextInt(3) - 1, 0, random.nextInt(3) - 1);
                    if (!world.canSetBlock(blockPos3)) {
                        return;
                    }

                    if (world.isAir(blockPos3.up()) && this.hasBurnableBlock(world, blockPos3)) {
                        world.setBlockState(blockPos3.up(), AbstractFireBlock.getState(world, blockPos3));
                    }
                }
            }
        }
    }
```
Here, `random.nextInt(bound)` generates a random number between 0 (included) and bound (excluded)\\
`this.hasBurnableBlock(world, blockPos3)` checks if the specified blockpos is a flammable block\\
`this.canLightFire(world, blockPos2)` checks if there is adjacent block of the given blockpos that is flammable.\\
`blockState.getMaterial().blocksMovement()` is false when it can not be pushed by a piston, no matter if it breaks (ex leaves) or stops (ex obsidian).

The key take away here is that the lava block randomly selects blocks, and checks if it is air, if so checks surrounding conditions to decide if it should be lit on fire: it is based on air blocks but not flammable blocks

## Fire Spread
```java
    
    @Override
    public void scheduledTick(BlockState blockState, ServerWorld serverWorld, BlockPos blockPos, Random random) {
        serverWorld.getBlockTickScheduler().schedule(blockPos, this, method_26155(serverWorld.random));
        if (serverWorld.getGameRules().getBoolean(GameRules.DO_FIRE_TICK)) {
            if (!blockState.canPlaceAt(serverWorld, blockPos)) {
                serverWorld.removeBlock(blockPos, false);
            }

            BlockState blockState2 = serverWorld.getBlockState(blockPos.down());
            boolean bl = blockState2.isIn(serverWorld.getDimension().getInfiniburnBlocks());
            int i = blockState.get(AGE);
            if (!bl && serverWorld.isRaining() && this.isRainingAround(serverWorld, blockPos) && random.nextFloat() < 0.2F + i * 0.03F) {
                serverWorld.removeBlock(blockPos, false);
            } else {
                int j = Math.min(15, i + random.nextInt(3) / 2);
                if (i != j) {
                    blockState = blockState.with(AGE, j);
                    serverWorld.setBlockState(blockPos, blockState, 4);
                }

                if (!bl) {
                    if (!this.areBlocksAroundFlammable(serverWorld, blockPos)) {
                        BlockPos blockPos2 = blockPos.down();
                        if (!serverWorld.getBlockState(blockPos2).isSideSolidFullSquare(serverWorld, blockPos2, Direction.UP) || i > 3) {
                            serverWorld.removeBlock(blockPos, false);
                        }

                        return;
                    }

                    if (i == 15 && random.nextInt(4) == 0 && !this.isFlammable(serverWorld.getBlockState(blockPos.down()))) {
                        serverWorld.removeBlock(blockPos, false);
                        return;
                    }
                }

                boolean bl2 = serverWorld.hasHighHumidity(blockPos);
                int k = bl2 ? -50 : 0;
                this.trySpreadingFire(serverWorld, blockPos.east(), 300 + k, random, i);
                this.trySpreadingFire(serverWorld, blockPos.west(), 300 + k, random, i);
                this.trySpreadingFire(serverWorld, blockPos.down(), 250 + k, random, i);
                this.trySpreadingFire(serverWorld, blockPos.up(), 250 + k, random, i);
                this.trySpreadingFire(serverWorld, blockPos.north(), 300 + k, random, i);
                this.trySpreadingFire(serverWorld, blockPos.south(), 300 + k, random, i);
                BlockPos.Mutable mutable = new BlockPos.Mutable();

                for (int l = -1; l <= 1; l++) {
                for (int m = -1; m <= 1; m++) {
                for (int n = -1; n <= 4; n++) {
                if (l != 0 || n != 0 || m != 0) {
                int o = 100;
                if (n > 1) {
                    o += (n - 1) * 100;
                }

                mutable.set(blockPos, l, n, m);
                int p = this.getBurnChance(serverWorld, mutable);
                if (p > 0) {
                    int q = (p + 40 + serverWorld.getDifficulty().getId() * 7) / (i + 30);
                    if (bl2) {
                        q /= 2;
                    }

                    if (q > 0 && random.nextInt(o) <= q && (!serverWorld.isRaining() || !this.isRainingAround(serverWorld, mutable))) {
                        int r = Math.min(15, i + random.nextInt(5) / 4);
                        serverWorld.setBlockState(mutable, this.method_24855(serverWorld, mutable, r), 3);
                    }
                }
                }
                }
                }
            }
            }
        }
    }
    private void trySpreadingFire(World world, BlockPos pos, int spreadFactor, Random rand, int currentAge) {
        int i = this.getSpreadChance(world.getBlockState(pos));
        if (rand.nextInt(spreadFactor) < i) {
            BlockState blockState = world.getBlockState(pos);
            if (rand.nextInt(currentAge + 10) < 5 && !world.hasRain(pos)) {
                int j = Math.min(currentAge + rand.nextInt(5) / 4, 15);
                world.setBlockState(pos, this.method_24855(world, pos, j), 3);
            } else {
                world.removeBlock(pos, false);
            }

            Block block = blockState.getBlock();
            if (block instanceof TntBlock) {
                TntBlock.primeTnt(world, pos);
            }
        }
    }

```

`serverWorld.getBlockTickScheduler().schedule(blockPos, this, method_26155(serverWorld.random))` schedules the scheduledTick method for a block at the given position to be called a certain amount of ticks later, given by `    private static int method_26155(Random random) {return 30 + random.nextInt(10);}`
# My solution
## Initial Idea and Naive Solution
If possible, a perfect way to standardize the woodlight time would to make it always light on the (statistically) expected time it take to light the portal. 

For this initial solution, to make everything simple, we are going to only consider the lava random ticks, but not fire spreading. It is also going to have many many problems, but we are going to solve it in the improved methods.

The core idea to this solution, is that we are going to change the random tick into ticking every single block in every subchunk (again, we are going to change this later). Then, everything is going to instantly light, so we add a counter to each air block. When a lava potentially lights this block on fire, it now only adds to the counter, and the air is lit on fire only when the counter reaches a specific number. 

### Problems to solve
1. Obviously, resources to tick every block in the world every tick
2. Resources to create counters on each air block
3. Wood that works to light different spots in the portal do not work cooperatively
4. The counter starts couting even if can't form a portal, which should not be counted
5. Fire Spread
6. If you looked carefully at the code, there is more randomness within the random tick function

6 is easily solvable by making one lava increment different counters, potentially differently so that it matches the probability. The 
```java
else if (blockState.getMaterial().blocksMovement()) {
                        return;
                    }
```
shouldn't matter that much, since it needs `i=2` and the first try to not be air, the second try to be flammable space. If it really is a problem, it can be easily estimated by the number of nonmovable blocks in the `3x3` area above the lava.

## Computational Resouce Optimization
Since we only care about lava, the first thing we can do is move all of this functionality from random tick to scheduled tick, which is already a massive improvement since we are only ticking lava now. 

Now, you may think that there is still a lot of lava that generates in the world, so thats still a problem. However, we can create a way to only tick "relavent lava". Specifically, lava that has any obsidian around. You place obsidian in the world, it triggers a boolean flag and updates the lava. When the lava is updates, and the flag is true, then it triggers counters and starts continuously scheduling a tile tick in the next tick, and repeats such a process until the flag is turned off. This flag is turned off when a lava block gets a special update called, and checks if there is any relavent obsidian around. If not, it turns off the flag. \\
> Note that both of these would be custom updates (kind of like comparator updates) since it as custom region and functionality.

Also, through this method, it is easy to exclude irrelavent lava generated in a ravine, since naturally generated obsidian wouldn't trigger such update.

## Too Many Counters
My solution to the fact that there are too many counters, is to move the counter to the obsidian. When an air block gets triggered, it searches downward for obsidian, and adds to that counter. This obsidian is also responsible to checking if it is the bottom of a full portal: if not then it rejects this increment to the counter; if so it also checks the sum of all counter for this same portal.

> Notice that this solution also solves problem 3 and 4.

## Fire Spread
Fire spread is way harder to fix, since problem 3 is always glaring. However, this is the only reason leaves are faster than wood. This time i would actually recommend  to just approximate a boost for the counters if you use leaves since the ranges are so alike and the choice is often trivial. I believe that this wouldn't change the stragey, unless with further research, which can also guide to fix the amount of boost.

## Further potential problems
- As mentioned, further research in firespread stuff
- Damage that fire does on players
- Many tradebacks between performance and accuracy, only decidable after testing

# Some solutions i don't like and why
- Comming up with a formula of the amout of wood, leaves, and lava around the portal\
I think this is just a little too foolable, by spamming the requirements
- Basically any similar idea, of ranking a block by how helpful it is to the woodlight, as it is always going to be more exploitable. 
> (I think ideally no "multilock structure recognizing" beyond recognizing an ignitable protal should be used)
