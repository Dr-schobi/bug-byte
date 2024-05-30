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


## my solution with C and a little ++

I decided to use a compact internal representation as a 24 length char array.
This should be plenty to store all weights (no numbers >200, no negative numbers).

### define the input

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

I also need to define the edges of the graph. 
Each edge is identified by two node names and a weight. I just put zero if it is not yet set.
Ideally, I would like to store a node as chars `{1, 2, 5}` which will be a pain to read.
I chose to write down the asci character numbers of the names like this:

```c
int numberofconns=24;
char conns[24][3] = { {'A', 'B', 0},
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
I used this function a lot for debugging. With `pos` being set to a position, I can also illustrate where we are currently. For `pos=2` this would give some lines around 'BD': 

```
AB AC|BD|CD EF CF KF FH GH HI KH MH DI IJ MI LK OK KP MN OP PQ MQ QR RO
 0  0| 0|12  0  0  0  0  0  0 24  0  0  0 20  0  7  0  0  0  0  0  0  0
```








