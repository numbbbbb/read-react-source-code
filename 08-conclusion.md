# Conclusion

Although I don't like React itself, I really love reading its source code.

At the beginning, I have no idea where to start with and where should I go next. I was stuck many times and must try my best to go on. It's really hard, but really fulfill me. I love the challenge.

We start with the build process. At this time React use three different tools to build it. It's tricky and interesting to see how to combine them together.

Then we go to render process. This is the first React call for everyone. We meet batch updating, injection, and transaction for the first time. During the tracing, we jump between many files, this is a good train to learn how to split different parts into different files. We also learn how to use injection for extracting implements and make the core part more generic.

Next, we have a long journey to setState. We go into details of batch updating, pooledclass, and transaction. They are designed to make the process faster, more flexible and more controllable. Read them carefully, you can learn many practicable patterns and skills.

After that, we learn how to understand other functions based on what we already know. You can find everything during render and setState process. Then just trace it and read whatever you meet.

Finally, we compare React and Vue. Maybe you don't agree with me, but I hope you can gain some new ideas from that.

In short, I hope to show you **how to read the source code**. The most important thing is the methods and skills I used, not the explanation I said.

Hopes this series can be useful to you. And if you find some mistakes, feel free to give me a feedback.

Thank you.

