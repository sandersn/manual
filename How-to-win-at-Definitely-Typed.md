# How to win at Definitely Typed

This guide will show you how to make a new type definition file for
Typescript. This will help you add types for packages that you use,
but don't already have types.

I'll use estraverse for this; once youve seen how its done, you can add types for esrecirse as an exercise to make sure you understand how to do it yiurself. if you want to follow along, then you will want to clone both estools/estraverse ad DefintielyTyoed/DefinitelyTyped. The, open estraverse.js.  

## Structure

the furst step to a non-blank defintion file is the moduke structure. There is a good oage on this in the handbook.  Bascially, it boils down to figrung out how the moduke exoorts its contents and, conversly, how people will import the modukes contents. 

tyoescrupt expresses moduke imports in Es6 syntax, pkus an extension fir commonjs-style module.exports assignment. Since estraverse uses commonjs, youll have to map the commonjs exoorts to ES6 syntax. by searchi g fir the word esports, i can quickly see that the translation wint be too hard:

exports.traverse = traverse etc

will transkate to

export { traverse }

or you could write it as 

export function ttaverse { 

durectly on the original deckaration


## Cheating

You can cheat with a single%line declare module statement. You can also feckare almost anythin as any and move on with your life. 


haha wow my typing withnout autocorrect is really bad  
