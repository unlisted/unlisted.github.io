---
layout: post
title:  "A State Machine"
date: 2017-04-05 
categories: c++
---

Design and initial implementation general purpose state machine in c++. 


### Motivation ###
The need to understand current state and define explicit transition path comes up often enough to warrant spending some time on a general solution. It's also a nice problem to give to candidates during interview. 

### Requirements ###
1. Should be easy to add to existing objects.
2. Need to be able to use custom states (start, stop, cancel, etc...)
3. Need to support arbitrary transitions from between states.
4. Should be able to define transition rules.
5. Success should be optionally predicated on result of arbitrary function.

### Design ### 
Plain old c++ inheritance is good candidate for implementation. We want only the object that we giving statefulness to be able to access the internal functions. We should restrict external access to things like current state.

The inheriting class needs to handful of simple of functions: 
1. A function to add new transition which should take a command, a pairs of states (initial and final), and a transition function.

2. An init function to setup the initial state.

3. A function for transitioning from one state to another.
    1. Should check if the desired state exists.
    1. Should check if the desired transition is valid given the current state and desired state.
    2. If the transition function is provided (not nullptr perhaps), it's return value should be checked before transitioning. Can be used for error handling, transition failure, etc...

### Implementation ###
States and commands can be simple enums for this first implementation. 
{% highlight c++ %}
 7 enum State {
 8     initialized,
 9     idle,
10     running,
11     finished,
12     cancelled,
13     error
14 };
15
16 enum Action {
17     start,
18     stop,
19     cancel,
20     reset,
21     finish
22 };
{% endhighlight %}
Also for the first implementation we can use a few unordered_maps to store valid commands, transitions and transition functions.
{% highlight c++ %}
44 using Transition = std::pair<State, State>;
45
46 // State: Current State
47 // Actions: Allowed Actions
48 using ActionMap = std::unordered_map<State, std::vector<Action>>;
49
50 // Action: Desired action
51 // Result: State resulting from action
52 using ResultMap = std::unordered_map<Action, State>;
53
54 // Action: Desired Action
55 // Function: State transition success depends on result
56 using FunctionMap = std::unordered_map<Action, std::function<bool()>>;
{% endhighlight %}
In order to use enums as keys for our unordered_maps we need provide specializations for hash in std namespace
{% highlight c++ %}
 24 namespace std {
 25
 26     template<>
 27     struct hash<Action> {
 28         std::size_t operator()(const Action a) const
 29         {
 30             return static_cast<std::size_t>(a);
 31         }
 32     };
 33
 34     template<>
 35     struct hash<State>
 36     {
 37         std::size_t operator()(const State s) const
 38         {
 39             return static_cast<std::size_t>(s);
 40         }
 41     };
 42 }
{% endhighlight %}
Values for these maps need to be provided to AddTransition function for each desired transition. This implementation works, but it's a bit clunky. We'll revisit in the future.
{% highlight c++ %}
 9 void StateMachine::AddTransition(Action action, Transition transition, const std::function<bool()> &func)
10 {
11     // get ref to actions allow from initial state.
12     // accessor creates if not exist.
13     auto& actions = _actionMap[transition.first];
14
15     // add action to list
16     actions.push_back(action);
17
18     // set result
19     _resultMap[action] = transition.second;
20
21     // set transition function
22     _funcMap[action] = func;
23
24 }
{% endhighlight %}
Only the desired state is provided as a parameter to the function which handles transitions. The implementation should check if the transition exists, if the transition is valid and if the transition is successful.
{% highlight c++ %}
50 bool StateMachine::DoTransition(Action action)
51 {
52     // check if desired state exists
53     auto found = _resultMap.find(action);
54     if (found == _resultMap.end())
55         return false;
56
57     // check if transition is valid
58     auto valid = false;
59     auto& actions = _actionMap[_current];
60     for (auto& it : actions) {
61         if (it == action) {
62             valid = true;
63             break;
64         }
65     }
66
67     if (!valid)
68         return false;
69
70     // check result of function if one exists
71     if (_funcMap[action] == nullptr || _funcMap[action]()) {
72         _current = _resultMap[action];
73         return true;
74     }
75
76     return false;
77
78 }
{% endhighlight %}
Finally, we use accessors to give derived classes access to the internals. Only GetState is visible externally. Only the derived class should be able to manipulate it's state. It can provide it's own public interface if necessary.
{% highlight c++ %}
 60 public:
 61     State GetState() { return _current; }
 62
 63 protected:
 64     StateMachine();
 65     virtual ~StateMachine();
 66     bool Init(State initial_state);
 67     void AddTransition(Action action, Transition transition, const std::function<bool()> &func = std::function<bool()>(nullptr));
 68     bool DoTransition(Action action);
{% endhighlight %}
Example of usage.
{% highlight c++ %}
#include <iostream>
#include <memory>
#include <string>
#include "state_machine.h"

using namespace std;

class StatefulObject: public StateMachine
{
public:
    StatefulObject()
    {
        AddTransition(start, make_pair(idle, running), []{return true;});
        AddTransition(stop, make_pair(running, idle), []{return true;});
        AddTransition(cancel, make_pair(idle, cancelled), []{return true;});
        AddTransition(reset, make_pair(cancelled, idle));

        Init(idle);
        cout << GetState() << endl;

        DoTransition(start);
        cout << GetState() << endl;

        DoTransition(stop);
        cout << GetState() << endl;

        DoTransition(cancel);
        cout << GetState() << endl;

        DoTransition(reset);
        cout << GetState() << endl;
    }
};


int main(int argc, char* argv[])
{
    StatefulObject rs;
	return 0;
}
{% endhighlight %}
And the output

    morgann@toaster:~/work/unlisted.github.io$ ~/work/c++/state_machine/build/bins/sm
    1
    2
    1
    4
    1

Here's a link to the souce [code](https://github.com/unlisted/state_machine).

And here are a few thoughts about what can be improved.
1. Better interface for adding transtions
2. Error/exception handling
3. Ability to Remove transition
4. Transition log (track transitions through process)
5. Support concurrency.

