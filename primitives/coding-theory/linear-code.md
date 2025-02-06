# Linear Code

If the linear combination of codewords is also a codeword, the code is called a [linear code](https://en.wikipedia.org/wiki/Linear_code).

$$
c_1, c_2 \in C \land a \cdot c_1 + b \cdot c_2 \in C
$$

where $$a, b \in \mathbb{F}_q$$.

**Reed-Solomon (RS) codes**, which are commonly used, satisfy this property and are therefore linear codes. (This property allows the folding operation to be used in RS-based FRI.)

$$
\mathsf{Enc}(f_1) = \{f_1(\omega), f_1(\omega^2), f_1(\omega^3), f_1(\omega^4)\} \\
\mathsf{Enc}(f_2) = \{f_2(\omega), f_2(\omega^2), f_2(\omega^3), f_2(\omega^4)\} \\ 
\mathsf{Enc}(a\cdot f_1 + b \cdot f_2) = 
\Big\{ 
\begin{aligned}
a\cdot f_1(\omega) + b\cdot f_2(\omega), a\cdot f_1(\omega^2) + b\cdot f_2(\omega^2), \\ a\cdot f_1(\omega^3) + b\cdot f_2(\omega^3), a\cdot f_1(\omega^4) + b\cdot f_2(\omega^4)
\end{aligned}
\Big\}
$$

The traditional encoding function used in RS-codes involves FFT operations. However, due to the limitations mentioned earlier, alternative encoding methods must be explored.

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
