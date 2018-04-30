
1. $\textit{post}(P_1;P_2, s) \triangleq \textit{post}(P_2,\textit{post}(P_1,s))$
2. $\textit{post}(v_1 \gets x) \triangleq s[v \mapsto s(x)]$
3. $\textit{post}(v_1 \gets c \times v_2) \triangleq s[v \mapsto c \times s(v_2)]$
4. $\textit{post}(v_1 \gets v_2 + v_3) \triangleq s[v \mapsto s(v_1) + s(v_2)]$
5. $\textit{post}(v_1 \gets sin(v_2)) \triangleq s[v \mapsto sin(s(v_2))]$


Now, forward differentiation:

1. No change.
2. $\textit{post}(v_1 \gets x) \triangleq s[v \mapsto x][\dot v \mapsto 1]$
3. $\textit{post}(v_1 \gets c \times v_2) \triangleq s[v \mapsto c \times s(v_2)][\dot v \mapsto c * \dot v_2]$
4. $\textit{post}(v_1 \gets v_2 + v_3) \triangleq s[v \mapsto s(v_1) + s(v_2)][ \dot v \mapsto s(\dot v_1) + s(\dot v_2)]$
5. $\textit{post}(v_1 \gets sin(v_2)) \triangleq s[v \mapsto sin(s(v_2))] [\dot v \mapsto \dot v_2 \times cos(s(v_2))]$
