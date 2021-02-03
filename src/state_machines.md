# State Machines

Access control can be viewed as a special case of a **state machine**. For example, a mutex is a two-state machine:

<center>
<svg width="522" height="155" version="1.1" xmlns="http://www.w3.org/2000/svg">
	<ellipse stroke="black" stroke-width="1" fill="none" cx="473.5" cy="70.5" rx="47" ry="47"/>
	<text x="446.5" y="76.5" font-family="Menlo, monospace" font-size="16">locked</text>
	<ellipse stroke="black" stroke-width="1" fill="none" cx="209.5" cy="70.5" rx="47" ry="47"/>
	<text x="172.5" y="76.5" font-family="Menlo, monospace" font-size="16">unlocked</text>
	<path stroke="black" stroke-width="1" fill="none" d="M 437.283,100.248 A 180.958,180.958 0 0 1 245.717,100.248"/>
	<polygon fill="black" stroke-width="1" points="245.717,100.248 249.857,108.725 255.151,100.241"/>
	<text x="275.5" y="148.5" font-family="Menlo, monospace" font-size="16">Mutex::unlock()</text>
	<polygon stroke="black" stroke-width="1" points="117.5,70.5 162.5,70.5"/>
	<text x="0" y="76.5" font-family="Menlo, monospace" font-size="16">Mutex::new()</text>
	<polygon fill="black" stroke-width="1" points="162.5,70.5 154.5,65.5 154.5,75.5"/>
	<path stroke="black" stroke-width="1" fill="none" d="M 249.513,46.021 A 215.88,215.88 0 0 1 433.487,46.021"/>
	<polygon fill="black" stroke-width="1" points="433.487,46.021 428.38,38.089 424.119,47.135"/>
	<text x="285.5" y="16.5" font-family="Menlo, monospace" font-size="16">Mutex::lock()</text>
</svg>
</center>

State machines have two core concepts: **states** (the circles) and **transitions** (the arrows). When APIs represent state machines, the important question is whether the transitions are consistent with the states, e.g. you should not be able to unlock an unlocked mutex. Here are a few more examples of state machines in systems:

* A shopper can only checkout while their cart is not empty.
* A file can only be closed while its file descriptor is open.
* A [GPIO pin](https://rust-embedded.github.io/book/static-guarantees/state-machines.html) can only be written to when in write mode.
