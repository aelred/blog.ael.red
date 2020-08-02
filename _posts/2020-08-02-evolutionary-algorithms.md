---
title: Writing Hamlet with evolutionary algorithms
tags:
- evolutionary-algorithms
---

I discovered evolutionary algorithms in my teens and was fascinated. After a bit of research, I somehow managed to hack together a neural network in [GameMaker](https://www.yoyogames.com/gamemaker), attach it to the brains of a virtual car and watch it crash into walls.

I raced hundreds of these cars against each other. The best ones went on to the next round (with randomly generated modifications) and they all raced again. The outcome of this brutal process was a passable motorist, albeit not one likely to pass their driving test first time.

I was reminded of my obsession recently watching [Code Bullet](https://www.youtube.com/CodeBullet) demonstrating things I'd never heard of, such as NEAT and Q learning. Anyway, it got me wanting to implement this stuff again.

So let's get some definitions out of the way: an *evolutionary algorithm* is an algorithm inspired by biological evolution. It follows these steps:

1.  Generate a population of random individuals.
2. Repeat the following:
	1. Assess each individual based on how well it solves a problem (the _fitness_ of the individual).
	2. Breed a new population, based on _mutation_ and _crossover_ of the fittest individuals in the current population.

_Mutation_ refers to randomly altering an individual, such that it behaves slightly differently. _Crossover_ means taking two "parent" individuals and combining them to make a new "child" individual.

These two operations can improve fitness and maintain genetic diversity: mutation introduces diversity, allowing new solutions to be explored. Crossover avoids reducing genetic diversity by combining promising individuals together instead of allowing one fit individual to out-compete another.
# To be, or not to be
I'll use [Rust](https://www.rust-lang.org/) for my examples, but follow along in any language you like! The full code is available on [my GitHub](https://github.com/aelred/evolutionary-algorithms).

It's said that [if you put a monkey in front of a typewriter, given enough time it will produce the complete works of Shakespeare](https://en.wikipedia.org/wiki/Infinite_monkey_theorem).

We haven't infinite time (or easy access to monkeys!), so let's try something simpler, reproducing just this fragment from Hamlet:

```rust
const HAMLET: &str = "To be, or not to be--that is the question";
```

Our individuals are going to try their best to write Hamlet. So we'll represent them with strings:

```rust
#[derive(Debug)]
struct Individual(String);
```

Let's write out the basic algorithm:

```rust
// If you're unfamiliar with Rust, `usize` is just an integer!
const POPULATION_SIZE: usize = 1000;

fn evolve() {
    let mut population = gen_initial_population(POPULATION_SIZE);

    loop {
        population = breed_next_generation(population);
    }
}
```

Yes, that is an infinite loop! We'll come back to it later. Next we'll implement these:

```rust
fn gen_initial_population(size: usize) -> Vec<Individual> {
    unimplemented!()
}

fn breed_next_generation(population: Vec<Individual>) -> Vec<Individual> {
    unimplemented!()
}
```

# Generating an initial population

Our initial population can just be random strings with the same length as the target text: I counted and that's 41 characters!

```rust
const LENGTH_HAMLET: usize = HAMLET.len();

fn gen_initial_population(size: usize) -> Vec<Individual> {
    let mut population = Vec::new();

    for _ in 0..size {
        let mut individual = Individual(String::new());
        for _ in 0..LENGTH_HAMLET {
            individual.0.push(rand::random::<char>());
        }
        population.push(individual);
    }

    population
}
```

To generate random characters, we need to depend on `rand`:
```toml
# Cargo.toml
[dependencies]
rand = "0.7.3"
```

Let's generate three individuals:

```rust
fn main() {
    dbg!(gen_initial_population(3));
}
```

```bash
$ cargo run
[src/main.rs:4] gen_initial_population(3) = [
    Individual(
        "\u{495d5}\u{990aa}\u{ed152}톘\u{5a426}𭼟...\u{e7849}\u{477d8}",
    ),
    Individual(
        "\u{102f12}\u{760ff}\u{cdffa}\u{ee293}...𘉳\u{3ae05}퍬\u{1a846}",
    ),
    Individual(
        "\u{d78bb}𪶎\u{be97e}\u{41b4f}...\u{820da}\u{1084f8}\u{d4ea7}",
    ),
]
```

Eww... let's stick to printable ASCII characters:

```rust
use rand::Rng; // for `gen_range` method

fn gen_initial_population(population_size: usize) -> Vec<Individual> {
    ...
				
        for _ in 0..196 {
            individual.0.push(gen_ascii_char());
        }
				
    ...
}

fn gen_ascii_char() -> char {
    // printable ASCII ranges from 32 (space, ' ') to 126 (tilde, '~')
    rand::thread_rng().gen_range(32u8, 126u8) as char
}
```

```bash
$ cargo run
[src/main.rs:4] gen_initial_population(3) = [
    Individual(
        ",J>g\"\\dLPq9E{$JnIAO[;Q)0Zel):fM/Iu!fB)/Tu",
    ),
    Individual(
        "<?kO_gb|.SHw9ZAg=v{gfy4|JF/GG_X&/Sbw\'?PrJ",
    ),
    Individual(
        "Mu2``(w<.IG)cF)o@\\u:Tf f#ZNaOt5\'Vrg;Q$I`#",
    ),
]
```

Beautiful... kind of!

# Crossover and mutation

Now the meaty part, creating the next generation. We need to do three things:
1. Select the fittest individuals in the current population
2. Generate new individuals by crossover
3. Randomly mutate each new individual

```rust
fn breed_next_generation(mut population: Vec<Individual>) -> Vec<Individual> {
    select_fittest_individuals(&mut population);
    let mut children = crossover_population(&population);
    mutate_population(&mut children);
    children
}
```

Let's select the fittest half of the population:

```rust
fn select_fittest_individuals(population: &mut Vec<Individual>) {
    population.sort_by_cached_key(|i| -i.fitness());
    population.truncate(population.len() / 2);
}
```

Our fitness function is simply "how similar is it to `HAMLET`"? We can count how many characters match those in `HAMLET`:

```rust
impl Individual {
    fn fitness(&self) -> i32 {
        let my_chars = self.0.chars();
        let target_chars = HAMLET.chars();
        let chars_same = my_chars.zip(target_chars).filter(|(x, y)| x == y);
        chars_same.count() as i32
    }
}
```

The basics of crossover are, we randomly select two parents and combine them to make a new individual, repeating until we have a full population:

```rust
fn crossover_population(parents: &[Individual]) -> Vec<Individual> {
    let mut population = Vec::new();

    use rand::seq::SliceRandom; // for `choose` method
    let mut random = rand::thread_rng();

    for _ in 0..POPULATION_SIZE {
        let parent1 = parents.choose(&mut random).unwrap();
        let parent2 = parents.choose(&mut random).unwrap();
        let child = parent1.crossover(parent2);
        population.push(child);
    }

    population
}
```

Crossover of individuals is interesting. We can do it in a few ways. I'm going with [single-point crossover](https://en.wikipedia.org/wiki/Crossover_(genetic_algorithm)#Single-point_crossover). We randomly pick an index in the string: everything left of that index comes from the first parent, everything to the right comes from the second. For example:

```
    CROSSOVER POINT |
                    |
parent1: "hello I am|a string"
parent2: "yo it's me|parent 2"
                    |
child:   "yo it's me|a string"
```

Here's our implementation:

```rust
impl Individual {
    fn crossover(&self, other: &Individual) -> Individual {
        let index = rand::thread_rng().gen_range(0, LENGTH_HAMLET);

        let left = self.0[0..index].chars();
        let right = other.0[index..LENGTH_HAMLET].chars();
        let child_string = left.chain(right).collect();

        Individual(child_string)
    }
}
```

Like crossover, there are many ways to do mutation. Let's pick something simple: each character has a 1% chance to be mutated.

```rust
fn mutate_population(population: &mut [Individual]) {
    for individual in population {
        individual.mutate();
    }
}

impl Individual {
    fn mutate(&mut self) {
        let mut random = rand::thread_rng();

        self.0 = self.0.chars().map(|current_char| {
            if random.gen_bool(0.01) {
                gen_ascii_char()
            } else {
                current_char
            }
        }).collect()
    }
}
```

# Unleash the monkey!
Now for some finishing touches... we'll print the fittest individual of each generation, and add our termination condition:

```rust
fn main() {
    evolve();
}

fn evolve()  {
    let mut population: Vec<Individual> = gen_initial_population(POPULATION_SIZE);

    loop {
        population = breed_next_generation(population);

        let fittest = population.iter().max_by_key(|i| i.fitness()).unwrap();
        println!("{}", fittest.0);

        if &fittest.0 == HAMLET {
            return;
        }
    }
}
```

And we can run it:

```
$ cargo run
&( R>ngurk>Oug}l V5*KSbE;&(bC&?m7JKe+A[#m
r?nb4?iiY,O<2\pk 1eJe}w6+t6p*.2e`2umRG{Pq
TDbGj%L+Z_><2\pk 1eJe}w6+t6p*.2e`2umRG{Pq
...
To bx,Zor}no+ 9o $0--t ab .smt72 quXst'on
To pe, or.nf# p0 be4-t ab i? 72e quesVion
To Re, or not 9o $e--twQr is`t!l quXsti+n
...
To be, or not to be--t]at is the question
To be, or not Oo be--that is the question
To be, or not to be--that is the question
```

Pretty quick too!

# Is this... useful?
I have no idea! This is my hobby. Certainly, ones as simple as this won't get you very interesting results, but the great thing about this algorithm is it's so general.

Just about every part of this algorithm can be changed:

- The fitness function could put an individual in a complex simulation, or test against a huge amount of training data, or even have individuals directly compete.
- Selection in each generation can retain the fittest individuals unchanged ([elitism](https://en.wikipedia.org/wiki/Genetic_algorithm#Elitism)).
- Way too many options to list for mutation or crossover.

Most importantly, what we produced above was a [genetic algorithm](https://en.wikipedia.org/wiki/Genetic_algorithm), these are evolutionary algorithms
where individuals are fixed-length strings. But there's no need to limit yourself to that! Instead of strings, we could encode individuals as:

- [Classifiers](https://en.wikipedia.org/wiki/Learning_classifier_system)
- [Neural networks](https://en.wikipedia.org/wiki/Neuroevolution)
- [Actual computer programs!](https://en.wikipedia.org/wiki/Genetic_programming)

I'm especially interested in neuroevolution - stay tuned for a post about that in the next decade 😃
