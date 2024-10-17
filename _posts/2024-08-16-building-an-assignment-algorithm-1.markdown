---
title: Building an Assignment Algorithm - Episode 1 / 3
date: 2024-08-16 9:22:00 Z
categories:
- Algorithms
tags:
- Algorithms
summary: How we built an assignment algorithm, the first blog in a series of 3.
author: jwarren
---

<!-- from: 2023-11-24-llm-mem.md -->
<style> 
    summary {
        font-weight: 300;
        display: block;
    }
    summary::after {
        cursor: pointer;
        content: '[+more]';
        text-decoration: underline;
        text-decoration-style: dotted;
        padding-left: 0.5em;
        font-size: 0.8em;
    }
    details[open] > summary::after {
        content: ' [−less]';
    }
</style>

Last year, our team was working on an app that organised conferences. Our most interesting mission, in my opinion, was to design and build an algorithm that assigned talks to attendees according to their choices. This algorithm would save organisers the time, human error and brain power required to ensure all attendees are fairly allocated. After having built and run our algorithm, we achieved results that improved the fairness of previously time-costly hand-calculated assignments by 30% (according to our measure), only taking seconds of calculation!

## The brief

The conference would be split into time slots where multiple talks take place in each slot and have attendees who order their preferences for which talks they would like to attend for each time slot. We needed to make an algorithm which would…

1. Take these choices, assign a talk to each attendee for every time slot 
2. Make assignments in the fairest way possible (minimising the chance of a user getting a 2nd or 3rd choice). 
3. Account for a minimum and a maximum capacity of each talk
4. Account for talks repeated in different slots (if they were believed in advance to be popular). i.e. Attendees shouldn’t be given the same talk twice.

## Genesis: a single time slot
We began by isolating the problem of how to assign talks for just one time slot, which  is a set of concurrent talks. However before we get bogged down in technical details, let's start with an example. 

An intergalactic 3-talk conference for universal problems has 5 attendees:

A. Anakin
B. Boba fett
C. Chewbacca
D. Darth Maul
E. Emperor Palpatine

The three talks run concurrently (denoted T1, T2, T3):

 - The Dark Side of Work-Life Balance: Avoiding Burnout in the Empire (T1)
 - Lightsaber Practise and Etiquette: Health and safety in the workplace (T2)
 - Parenting 101: how to tell a son that you’re his father (T3)

T1 has a maximum capacity of 3 attendees, the other two talks have a maximum capacity of 1.

Each attendee makes an ordered list of the talks. For example Anakin’s first choice is “Parenting 101”, his second choice is “The Dark Side of Work-Life Balance” and third choice is “Lightsaber Practise and Etiquette”. We can represent this as [1st: T3, 2nd: T1, 3rd: T2], or even more simply [3,1,2]. If we do the same for the other attendees, we have:

<center> A - [3,1,2] B - [1,3,2] C - [1,2,3] D - [1,3,2] E - [3,1,2]</center>
<br>
![fig1: The attendees with their different choices]({{ site.github.url }}/jwarren/assets/assignment-algorithm-1/characterChoices.png)
*fig1: the attendees choices represented graphically.*

<figure style="float: left; width: 30%; margin-right: 30px;">
    <img src="{{ site.github.url }}/jwarren/assets/assignment-algorithm-1/initialGroupingColour.png" alt="fig2: attendees grouped according to their first choice." style="width: 100%;">
    <figcaption><em>fig2: attendees grouped according to their first choice. We will come back to this layout in future examples.</em></figcaption>
</figure>

You can see this information explained in fig1 and fig2.

## Basic solution

Let’s begin with a simple solution. We go through the list of attendees alphabetically and assign each attendee their first possible choice:

1. Anakin gets his much needed first choice on parenting (T3).
2. Boba-Fett gets his first choice on avoiding burn-out (T1).
3. Chewbacca has his first choice on work life balance (T1).
4. Darth Maul can’t get his first choice of parenting (T3), because the last space was taken by Anakin. So he gets his second choice on work life balance (T1).
5. Emperor Palpatine can’t get his first choice of avoiding burn-out in the empire (T1), nor his second choice for parenting (T3). So he gets his third choice and so has to sit through a painfully boring health and safety talk. Heads will roll.

In summary:

1. A - T3 (1st)
2. B - T1 (1st)
3. C - T1 (1st)
4. D - T1 full, gets 2nd choice T3
5. E - T3 full, 2nd choice T3 full, 3rd choice T2

A second choice and a third choice is not ideal among 5 attendees. Of course the ordering here is contrived, but how would we process the assignments for an optimal result, no matter the situation?

We can solve one problem by making sure people who don’t get their first choice are more likely to get their second choice. For this next example, we will put attendees into groups according to what their first choice is as shown in the diagram above. If a talk is full, then we move the attendees to a new group according to their next choice. Whenever an attendee is moved between a group, we sort the unconfirmed attendees of that group according to what choice that group is for the attendee.
A choice is cemented if at that stage, every group can accept the top attendee. i.e. The group is not full. (As the diagram will show). Here’s an example of this in practice.

### Sort by choice example
<div style="position: relative; width: 100%; height: 0; padding-top: 100.0000%;
 padding-bottom: 0; box-shadow: 0 2px 8px 0 rgba(63,69,81,0.16); margin-top: 1.6em; margin-bottom: 0.9em; overflow: hidden;
 border-radius: 8px; will-change: transform;">
  <iframe loading="lazy" style="position: absolute; width: 100%; height: 100%; top: 0; left: 0; border: none; padding: 0;margin: 0;"
    src="https:&#x2F;&#x2F;www.canva.com&#x2F;design&#x2F;DAGNXzlAgEI&#x2F;AbkC2uJ3dy-q513EnbhUuA&#x2F;view?embed" allowfullscreen="allowfullscreen" allow="fullscreen">
  </iframe>
</div>

Oh no, the same thing happened again! This time the Emperor got his second choice, but Darth Maul got his third. Heads still may roll! Are there any solutions with no 3rd choice, you may ask? Chewbacca and the Emperor swapping talks. Both would get their second choice, no one would get their 3rd choice.

But how can we capture this in a rule? Chewbacca’s second choice (T2, Health and Safety) was not popular so had more capacity than the Emperor's second choice (T1 Work-life Balance) which was very popular. So let’s factor that in. Let’s say that when we order people, we sort according to choice and also by what the attendees next choice’s capacity is. We call this sorting by surplus difference - the score by which we measure how much free capacity an attendee's next choice has.
Essentially we are thinking ahead by saying people who have a very popular second choice will be more likely to get their first choice. Let’s look at another example with this factored in.

### Sort by surplus difference example
<div style="position: relative; width: 100%; height: 0; padding-top: 100.0000%;
 padding-bottom: 0; box-shadow: 0 2px 8px 0 rgba(63,69,81,0.16); margin-top: 1.6em; margin-bottom: 0.9em; overflow: hidden;
 border-radius: 8px; will-change: transform;">
  <iframe loading="lazy" style="position: absolute; width: 100%; height: 100%; top: 0; left: 0; border: none; padding: 0;margin: 0;"
    src="https:&#x2F;&#x2F;www.canva.com&#x2F;design&#x2F;DAGNXzlAgEI&#x2F;AbkC2uJ3dy-q513EnbhUuA&#x2F;view?embed" allowfullscreen="allowfullscreen" allow="fullscreen">
  </iframe>
</div>

It works! Balance in the force has been restored. 
Sorting by choice (first, second or third) is a way of avoiding a further need to compromise in the immediate present. It is a short term consideration. However, as we saw, only taking this short-term idea into account may not lead to a better outcome. For this reason, we introduced an ordering by surplus difference. Ordering by surplus difference is essentially looking ahead and avoiding attendees having to compromise in the future. In the context of a single time slot, it is a (relatively) long term consideration. Taking both the short term and the long term considerations into account requires a balance as they need to be ordered simultaneously. 
<details><summary>If you would like to know how we calculated these values, click the 'more' button for more details</summary>
Surplus difference is calculated by finding the difference between the room surplus of the current choice and the room surplus of their next unassigned choice. Room surplus is calculated as follows:

<table>
  <tr>
      <th>Talk is…</th>
      <th>Room surplus equals…</th>
      <th>Room surplus range of values…</th>
  </tr>
  <tr>
    <td>Oversubscribed</td>
    <td>Number of group attendees - max attendees</td>
    <td>Room surplus > 0</td>
  </tr>
  <tr>
    <td>Undersubscribed</td>
    <td>Number of group attendees - min attendees - number of attendees in slot</td>
    <td>Room surplus < -(Number of attendees in the slot)</td>
  </tr>
  <tr>
    <td>Undersubscribed</td>
    <td>Number of group attendees - max attendees</td>
    <td>-(Number of attendees in the slot) < Room surplus < 0</td>
  </tr>
</table>

To account for the attendee choice, surplus difference would be multiplied by a weight, defined as so:

current group weighting = 2 - 1 /over (choice for their current group)
The weight gives more emphasis to the surplus difference if a user’s current group is their second choice, over whether a user is in their 1st choice. The formula given is if the user is on their first choice, the weight will be 1, if the user is on their 2nd choice, the weight will be 1.5 and if the user is on their 3rd choice, the weight will be 1.6666... The thought behind this was that though the “felt“ difference for the first few choices would be important to the user, the remaining choices would be marginally worse but have diminishing importance between them.
</details>
\
So you may be wondering, how do we define and measure compromise? And how can we do this over the course of multiple slots? These are good questions and will be answered in the next blog in the series. We’ll also get into the nitty gritty of the maths behind it all. Stay tuned!
