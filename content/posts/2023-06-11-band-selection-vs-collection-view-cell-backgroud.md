+++
title = 'UIBandSelectionInteraction vs. UICollectionViewCell background color'
date = '2023-06-11'
draft = false
+++

One of my colleagues (kudos, Nik) added the proper support of the pointer at iPad OS during our Ship It days (approximately once a month we can do any feature/improvement we want). After some time the team jumped in and we all were preparing it for release.

When it came to QA, we've got a strange ticket: file list view in Select mode would sometimes deselect all previously selected items. I drew the stick for this ticket and dived into it. The reason for the bug was obvious at first glance – band selection (`UIBandSelectionInteraction`) triggered 1x1 selection at the *tapped* cell, which did what it had to do – cleared the previous selection and selected the cells (that single one) that were intersected by the `selectionRect`.

<!--more-->

The bug was not that easy to reproduce. Sometimes I could select like 10 files, sometimes it was just two.

My first idea was... roll my eyes and tell,– "Oh, that Apple...". I investigated it for a while and didn't find a way to prevent `UIBandSelectionInteraction` from such behaviour. So, there was nothing else, but to implement my own UIBandSelectionInteraction. I looked at the Files.app and started making a list of the requirements:
 - basic selection
 - shift + selection
 - cmd + selection
 - autoscroll
 - the list went on and on

The day came to an end, and so did my cognitive abilities.

In the morning I decided to take a closer look at the surroundings in assembler. After a while, I found that band selection checks if `UIContol` subclass is at the start point.

I decided to remove all UIContols from the cell. And yay! the bug became 100% reproducible!

I checked Files app again – they didn't have such a bug. I checked a different collection view in our app – no such bug either. But what the actual hack?

It always works better when you walk away from your desk for a while. During a walk, I've got a new idea. What if it has something to do with the background color of our cell? I've painted it green – no bug, returned our color – have the bug. A-ha! But what's so special about our color? In Select mode we have a translucent gray frame around every selectable file. It's filled with our gray5 color – gray with 6 :) percent of alpha. I've played with that color and found out that as soon as alpha is < 10 % UIBandSelection thinks that the view is... transparent! But it's far from being transparent to the human eye!

I had a theory that maybe I could fool UIKit with not strictly gray color with the same alpha that we need, I mean, add 1 to any component in our 0x161616. No, that didn't work. Blue with alpha 5 is treated transparent too.

Ok, now I had all pieces of the puzzle. I had two options for the fix:
 1. ask our designers if we really needed that color to be 0.6 – wouldn't 10 work for us. It looked ugly, but I tried :)
 2. blend the cell background color with the collection view background and use that opaque color as the bg

You know the answer already. Now our selection file list selection color is opaque, and we have a long green comment explaining why that's so.
