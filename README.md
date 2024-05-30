# Jane Street Bug Byte Puzzle

Have you seen this puzzle? 
A number puzzle seemingly from April 2024 [https://www.janestreet.com/bug-byte/] 
This is my writeup of how I approached this puzzle.
I assume you are familiar with the instructions.
Spoiler warning: I won't post the solution, but I'm certainly going to reveal clues.

![I made a new drawing of this bug with names for the nodes](bug.svg)




## first observations:
- This looks quite unstructured - I gave the nodes some names with letters A through R. See my drawing above.

- some instrcutions where a little unclear (for me, as a non graph theorist):
  "non-self intersecing path from this node" seems to refer to nodes E, C, G and J.
  The route starts at each of these nodes, can go in any direction and ends when the sum is reached. It should not cross a node/edge twice. Looks like this excludes putting a 3 on edge GH and crossing this twice for obtaining the sum of 6.
    
- solving manually: At first, I thought that is something that can be solved by thinking about it.
  Around node B, there need to be numbers 1 and 2 to reach the sum of 3. 
  (Any number from 1-24 must be used only once and 0 is not allowed)
  Then edge AC can be 15 or 16. The condition for C to have a route to anywhere with sum 19 does not help here.
  I quickly lost track of which number I already spent.  It's time to kill it with iron - use more compute!

## try all combinations with python
In my day job, I'm working in python "all the time". 
But this is for glue code, bugfixing, for extending other peoples work. 
I've never worked with graphs in python before - but how hard can it be?
I represented the graph as a list of nodes and a list of edges/connections.
In the end I got this to evaluate about 10 combinations per msec on my old Celeron N4100.
Nothing to be proud of ...


## My solution with C and a little C++

Okay - let's try all combinations - how hard can it be?
I'll use a backtracking approach that puts a number, sees if it matches the conditions and trys the next one.
(Backtracking is nicely explained in the [Computerphile Sudoku solver](https://www.youtube.com/watch?v=G_UYXzGuqvM) video).


I decided to use a compact internal representation as a 24 length char array.
This should be plenty to store all weights (no numbers >200, no negative numbers).
I dont see a need to write C++ objects and abstractions. 
We've got maybe ~150 bytes to manage - if we dont waste, its going to be stay in the CPU chace and be faaaaaaast!


### setup

I'm using gcc with a makefile

```make
bug: bug.cpp
        gcc -Wall -O3 -o bug bug.cpp -lstdc++
```

and the following includes:
```c
#include <iostream>
#include <iomanip>
```



### Input definition - the nodes

For each node I need to know the desired sum of edges.
The node names 'A' to 'R' are simply the index into the array and `node[5]` will tell 
me that node F needs a sum of 54.
For `node[3]` there is 0 given (as there is no edge sum check possible).

```c
int numberofnodes=18;
char nodes[] = { 17, // A
                  3, // B
                  0, // C
                  0, // D
                  0, // E
                 54, // F
                  0, // G
                 60, // H
                 49, // I
                  0, // J
                 79, // K
                  0, // L
                 75, // M
                  0, // N
                 39, // O
                 29, // P
                 25, // Q
                  0}; // R
```
But how do I make sure that there is no typo in there?
Maybe print out the the nodes and double-check:

```c
// print node info 
void printnodes() {
   for (int i=0; i<numberofnodes; i++)
      std::cout << char(i+'A') << "=" << int(nodes[i]) << " ";
   std::cout << std::endl;
}
```
gives a list of node names with numbers - yes - this looks good!
```
A=17 B=3 C=0 D=0 E=0 F=54 G=0 H=60 I=49 J=0 K=79 L=0 M=75 N=0 O=39 P=29 Q=25 R=0
```

### Input definition - the edges

I also need to define the edges of the graph. 
Each edge is identified by two node names and a weight. I just put zero if it is not yet set.
Ideally, I would like to store a node as chars `{1, 2, 5}` which will be a pain to read.
I chose to write down the ASCII character numbers of the names like this:

```c
int numberofconns=24;
unsigned char conns[24][3] = { {'A', 'B', 0},
                               {'A', 'C', 0},
                               {'B', 'D', 0},
                               {'C', 'D', 12},
                               {'E', 'F', 0},
                               {'C', 'F', 0},
                               {'K', 'F', 0},
                               {'F', 'H', 0},
                               {'G', 'H', 0},
                               {'H', 'I', 0},
                               {'K', 'H', 24},
                               {'M', 'H', 0},
                               {'D', 'I', 0},
                               {'I', 'J', 0},
                               {'M', 'I', 20},
                               {'L', 'K', 0},
                               {'O', 'K', 7},
                               {'K', 'P', 0},
                               {'M', 'N', 0},
                               {'O', 'P', 0},
                               {'P', 'Q', 0},
                               {'M', 'Q', 0},
                               {'Q', 'R', 0},
                               {'R', 'O', 0} };
```
The result is a 2D char (byte) array with numbers for the ASCII chars from 'A' 65 and above. 
That is nice to read, but weird for processing. 
I put in some preprocessing whenever my programm starts and simply subtract 'A' (ASCII is soo nice! Dont ask me to make this UTF compliant!)  from each node like this:

```c
  // prepare correct indices, 'A' -> 0
   for (int i=0; i<24; i++) {
       conns[i][0] -= 'A';
       conns[i][1] -= 'A';
   }
```
So I finally have an array that tells me that edge 4 goes from this node number `conns[4][0]` to this node number `conns[4][1]` with the following weight `conns[4][2]`. There is 0 if no weight has been set yet.
And again, I need to check that this is correct:

```c
void printconns(int pos = 100) {

   for (int i=0; i<numberofconns; i++) {
       if ((pos == i) || (pos == i-1)) std::cout << "|";
       else std::cout << " ";
       std::cout << char(conns[i][0]+'A') << char(conns[i][1]+'A');
   }
   if (pos+1 == numberofconns) std::cout << "|";
   std::cout << std::endl;

   for (int i=0; i<numberofconns; i++) {
       if ((pos == i) || (pos == i-1)) std::cout << "|";
       else std::cout << " ";
       std::cout << std::setw(2) << int(conns[i][2]);
   }
   if (pos+1 == numberofconns) std::cout << "|";
   std::cout << std::endl;
}

```
The first part just prints the ABCs,
The second line then prints the current weight of the edge.

```
AB AC BD CD EF CF KF FH GH HI KH MH DI IJ MI LK OK KP MN OP PQ MQ QR RO
 0  0  0 12  0  0  0  0  0  0 24  0  0  0 20  0  7  0  0  0  0  0  0  0
```
Check successful!
I used this function a lot for debugging. With `pos` being set to a position, I can also illustrate where we are currently. For `pos=2` this would give some lines around 'BD'. Nice to watch when thousands scroll by. 

```
AB AC|BD|CD EF CF KF FH GH HI KH MH DI IJ MI LK OK KP MN OP PQ MQ QR RO
 0  0| 0|12  0  0  0  0  0  0 24  0  0  0 20  0  7  0  0  0  0  0  0  0
```

### Which numbers are available?

We are only allowed to use each number from 1-24 once. Somehow we need to keep track of this.
This is ripe for a off-by-one adressing problem - I'll better associate a 25 array for numbers that can be used 
('1' if number is available, '0' if number is not available).
Using a char for each number is a little excessive, as we would only need a single bit per number only. 
But using individual bits of a 32bit integer would be much more complex and likely not any faster.

```
char numbers[] = { 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
                   0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0};


// check which numbers are available
void updateavailablenumbers() {
    // clear availablenumbers
    for (int i=1; i<numberofnumbers; i++) {
        numbers[i] = 1;
    }
    // scan all conns
    for (int i=0; i<numberofconns; i++) {
        if (conns[i][2] > 0) {
            numbers[ conns[i][2] ] = 0;
        }
    }
    
    // analyze and print
    std::cout << "available numbers: ";
    int count=0;
    for (int i=1; i<numberofnumbers; i++) {
        if (numbers[i] == 1) {
            std::cout << i << " ";
            count++;
        }
    }
    std::cout << std::endl;
    std::cout << " (" << count << " total)" << std::endl;

}
```

At the start, this gives me the following list:

```
available numbers: 1 2 3 4 5 6 8 9 10 11 13 14 15 16 17 18 19 21 22 23 
 (20 total)
```
If you do not look closely, you will search for a while why this is showing all numbers and not working properly.
Only if you look closely, you can see that 7, 12, 20 and 24 are already missing from the list.

In principle, this function could be run for every step, over and over. 
I decided to keep track of numbers used and updated the available numbers array each time a number was used or given back. This gave another 5% speed increase.


### check node sums - O(n*m)

I need a function that goes through every node and sees if the sums are correct/exceeded already.
This is going to be a O(n*m) problem (n number of nodes, m number of edges), 
but with 18*24 this could still be okay-ish.
We have an iteration over all n nodes (and skip if there is no node-sum requirement for this node).
For each node we check all edges. 
If a node-number is used either as source `conn[i][0]` or as destination `conn[i][1]` (we don't care, our graph is undirected). We need to keep track in two ways:
- what is the sum for this node?
- do we still have a 0  weight in the edges? then we are not done and the sum will increase int he future

The second half of this function will analyze and print the results:
- allcomplete: all eddged for this node have a number assigned.
  - the computed sum is the correct sum as expected. good!
  - the sum does not match, we can abort. What we currently have is not a solution
- not all complete: this node has still some edges that are unkown and show 0
  - we expect the sum of all edges to this node to be smaller than the desired node sum. If so, ok - we need to wait.
  - if not, the sum is already too large and the can abort. Filling the other edges would only increase the sum later on. 

```c
bool checknodesums() {

    for (int node=0; node<numberofnodes; node++) {
        if (nodes[node] == 0) {
            // no requirement for this node
            continue;
        }
        int nodesum = 0;
        bool allcomplete = true;
        for (int i=0; i<numberofconns; i++) {
            if ((conns[i][0] == node) || (conns[i][1] == node)) {
                nodesum += conns[i][2];
                if (conns[i][2] == 0)
                    allcomplete = false;
            }
        }
        // std::cout << "node " << char(node+'A') << " = " << std::setw(2) << nodesum << " ";
        if (allcomplete) {
            if (nodesum == nodes[node]) {
                // std::cout << "    complete and ok" << std::endl;
            }
            else {
                // std::cout << "    complete but wrong sum!" << std::endl;             
                return false;
            }

        }
        else {
            if (nodesum < nodes[node]) {
                // std::cout << "not complete, still small enough" << std::endl;
            }
            else {
                // std::cout << "not complete, number too high!" << std::endl;          
                return false;
            }

        }

    }
    return true;
}
```

### check node sums - O(m+n)
It might be possible..?

### put a number and see if it works
Here is where the backtracking magic happens. 

I'll put the next available number and see if the sums are still good.
If yes - contine and put the next number (recursively).
If not, remove the number and try another one.



### results

On my outdated Celeron N4100 this runs in ~2.4 seconds.
Brute forcing this problem seemed to be okay.
In the end, the printing of millions of rows was sloing things down.

At first I got a larger list of matches. But remember? 
There were special conditions that I neglected.

With manual visual checking of the results, I got the following:
- as expected, A-B-D was always 1 and 2
- for edge G-H, among others, the solutions 3, 4 and 5 were proposed. But that would not fit the 6 "non-self-intersecing path" requirement. I added the condition to exclude 3, 5 and >6 for node GH.
- for edge J-I also got numbers >8. I added a condition that disallows numbers >8 for this node
- finally, I was down to two solutions that had E-f and C-F swapped. Only one of them matched the special conditions for nodes E amd C. 


## thoughts

- That was a fun exercise and I took a few detours in adding checks and more debugging.
  It is rare, that I'm able to spend a rainy day at the computer and dig deep into a challenge.
    

- I'm still repeating a lot of computation. 
  It was good to start this way and implement checks all over the place.
  
- That code might even run well on a microcontroller!

- I'm curious about other solutions and results. 
  I would be happy to see and discuss other solutions and I'm certainly willing to share my code!
  let me know of a [BRC](https://1brc.dev/) type of scoreboard.

Please contact me if anything is unclear 

