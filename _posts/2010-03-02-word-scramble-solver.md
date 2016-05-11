---
layout: post
title: How to Find Every Word in Word Jumble-Style Games 
summary: sucking the fun out of games, one game at a time :)
date: 2010-03-10
tags: java
---

Scramble is a word game where players try to find as many words as they can in a 4×4 or 5×5 grid of letters. I’ve been playing this game more than I’d like to admit to so in an attempt to hopefully ween me off the game, I’ve coded a solution in Java which finds every word in the grid.

![Scramble](/images/scramble.jpg)

### Setting Up the Problem

I set the problem up using an undirected, unweighted, connected graph.
Here’s my graph implementation using an adjacency list:

{% highlight java %}
import java.util.HashMap;
import java.util.HashSet;
import java.util.Set;


public class Graph<T> {
    private HashMap<T, HashSet<T>> adj;
    
    public Graph(){
        adj = new HashMap<T, HashSet<T>>();
    }
    
    public void addEdge(T from, T to){
        if(from != null && to != null){
            if(!hasVertex(from)) addVertex(from);
            if(!hasVertex(to)) addVertex(to);
            adj.get(from).add(to);
            adj.get(to).add(from);
        }
    }
    
    public void addVertex(T c){
        adj.put(c, new HashSet<T>());
    }
    
    public boolean hasVertex(T c){
        return adj.get(c) != null;
    }
    
    public Set<T> getVertices(){
        return adj.keySet();
    }
    
    public Set<T> getAdjacentVertices(T c){
        return adj.get(c);
    }
    
    public String toString(){
        String out = "";
        String graph = "";
        for(T c : adj.keySet()){
            out += c+":";
            for(T a : adj.get(c)){
                out += a+" ";
            }
            graph += out+"\n";
            out = "";
        }
        return graph;
    }
}
{% endhighlight %}

Each letter is represented by a `Node` object which is just a wrapper around a String.

To populate the graph, the game board is read from a file into a 2d array which is subsequently used to create the necessary vertices and edges:

{% highlight java %}
public static final int GAME_SIZE = 4;
public static final int BUFFER_SIZE = 2;
...
private Graph<Node> buildGraph(String file) throws Exception{
    Node[][] board = readBoardTo2DArray(file);
    Graph<Node> g = new Graph<Node>();
    for(int i = 1; i < GAME_SIZE + 1; i++){
        for(int j = 1; j < GAME_SIZE + 1; j++){
            Node from = board[i][j];
            g.addEdge(from, board[i-1][j]);
            g.addEdge(from, board[i-1][j-1]);
            g.addEdge(from, board[i-1][j+1]);
            g.addEdge(from, board[i+1][j]);
            g.addEdge(from, board[i+1][j-1]);
            g.addEdge(from, board[i+1][j+1]);
            g.addEdge(from, board[i][j-1]);
            g.addEdge(from, board[i][j+1]);
        }
    }
    return g;
}

private Node[][] readBoardTo2DArray(String file) throws Exception{
    Node[][] board = new Node[GAME_SIZE + BUFFER_SIZE][GAME_SIZE + BUFFER_SIZE];
    BufferedReader br = new BufferedReader(new FileReader(file));
    String line = null;
    int row = 1;
    while((line = br.readLine()) != null){
        String[] cline = line.split(" ");
        for(int i = 0; i < cline.length; i++){
            board[row][i+1] = new Node(cline[i]);
        }
        row++;
    }
    return board;
}
{% endhighlight %}

### Finding Every Word in the Graph

Now that we have a graph representation of the board, we can find every wordusing a variant of depth-first search.

{% highlight java %}
private Set<String> words;
...
public Set<String> findWords(){
    try{
        openConnection();
        HashMap<Node, Boolean> visited = new HashMap<Node, Boolean>();
        for(Node l : g.getVertices())
            visited.put(l, false);
    
        for(Node l : g.getVertices()){
            buildWords(g, l, visited, l.getValue());
        }
    }catch(Exception e){
        System.out.println(e.getMessage());
    }finally{
        try{
            closeConnection();
        }catch(Exception e){}
    }
    return words;
}

private void buildWords(Graph<Node> g, Node n, HashMap<Node, Boolean> visited, String word){
    visited.put(n, true);
    for(Node v: g.getAdjacentVertices(n)){
        String currWord = word + v.getValue();
        if(!visited.get(v)){
            visitWord(currWord);
            if(wordsMatch(currWord)){
                buildWords(g, v, visited, currWord);
            }
        }
    }
    visited.put(n, false);
}

private void visitWord(String word){
    if(isWord(word) && !words.contains(word)){
        words.add(word);    
    }
}
{% endhighlight %}

In findWords(), we initialize the visited HashMap which is used to keep track of the vertices we’ve seen on the current path and to prevent the same vertices from being used multiple times for the same word. From there, we call buildWords() with each of the vertices (treating each of the vertices as the root).
buildWords() constructs depth-first trees – Node n is the node we’re currently visiting and String word is the word that we’re constructing. First, we set the node to visited and then we travel down a path. To keep from proceeding down a path with no words, we apply a simple constraint to check that words exist with the letters on the current path. Without the constraint in place, the algorithm can be used to find every unique path in a graph.
Traditionally in DFS, we continually search deeper until a goal vertex is found or until we reach a terminal vertex with no children whilst marking vertices we’ve seen to keep from falling into infinite loops. Taking a coloring approach, we can color unvisited nodes white; visited nodes are colored two ways – gray vertices have some adjacent white vertices whereas all vertices adjacent to black vertices have been visited. By coloring vertices in this way, each vertex is visited at most once.
In our case, we don’t need to differentiate between visited vertices since rather than just traversing the graph to visit each vertex once, we want to potentially traverse every path in the graph. Fortunately for us, doing so requires only a single modification to DFS (asides from using a simpler coloring approach) – we reset the current vertex to not visited once we backtrace which allows for its inclusion in other paths.

### The Word List to Test Against
I used JDBC to connect to a mysql database built using the English word list from [ispell](http://wordlist.sourceforge.net/). I modified the word list by removing words less than 2 characters long and words with non-alphabetical characters. Unfortunately, I don’t know which word list is being used in Scramble and so many of the words found via my code are considered invalid by the game and vice versa.

### Conclusion
My solution takes <5 seconds to find all words in a game of Scramble and I'm guessing I could increase the performance by forking off calls to buildWords. Although it's a bit clumsy to use while playing since the games are timed, I managed to come in first a bunch of times while playing online multiplayer - my code has sufficiently sucked the fun out of the game and so hopefully I won't be playing it as much =).

All code is available [here](http://github.com/gurch101/ScrambleSolver). Note that a Java properties file (database.properties) is required with the following keys: jdbc.drivers, jdbc.url, jdbc.username, and jdbc.password. The jumble.txt file demonstrates the required format for the puzzle (space between each vertex; newline for each row).
