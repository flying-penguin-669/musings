# How frontier AI models learn

*Research synthesis · 20 September 2026*

**Frontier AI is improving by learning more reusable knowledge and procedures, learning which procedures succeed, and using more computation to solve each difficult problem.** Pretraining, reinforcement learning, expert data, and inference-time reasoning contribute to different parts of that process. Their interaction explains much more than treating them as competing ways to make a model “smarter.”

The central change in reasoning models is that we train a model to perform a sequence of useful computations before committing to an answer. That sequence can include inspecting evidence, trying an approach, checking it, and recovering from failure. Training improves the procedure; inference runs it on a new problem. Distillation can then make expensive learned behavior easier to reproduce.

This article presents a working explanation of those mechanisms and their implications. The linked studies support individual claims; the two frontier reports are examples of the mechanisms in deployed training pipelines.

## 1. A model needs knowledge, a procedure, and time to execute it

Consider a coding agent fixing an unfamiliar bug. It needs to understand programming and the relevant library; find the important files; form a hypothesis; make a change; run a test; and interpret the result. These are distinct requirements.

If it lacks the current source code, more thinking cannot reveal which change was committed yesterday. If it has the source but follows a poor debugging strategy, retrieval alone will not solve the problem. If it knows a good strategy but has time for only one guess, additional execution and checking can help.

**This gives us three useful questions: what information is missing, what procedure is missing, and what computation is missing?** There is also a fourth, commercial question: how much does obtaining a successful result cost?

These questions map onto the main interventions:

- **Pretraining** develops broad knowledge and reusable internal computations from large amounts of data.
- **Supervised fine-tuning (SFT)** teaches from examples: desired answers, reasoning steps, or action sequences.
- **RL** teaches from the consequences of sampled behavior, making rewarded decisions more likely across future tasks.
- **Context, retrieval, and tools** supply information and actions for the current task.
- **Inference-time reasoning and search** spend computation to work through that task.
- **Distillation** transfers behavior from a stronger or better-informed teacher into a student, reducing how much must be rediscovered or supplied repeatedly.

These are roles, not watertight compartments. Pretraining can learn procedures; SFT can teach knowledge; RL can learn new strategies. The distinction is how the learning signal is produced.

## 2. Pretraining learns machinery for using knowledge, as well as knowledge itself

Predicting the next token sounds shallow because the objective is simple. The computation that helps predict it need not be. To predict a line of code, a model benefits from tracking variables and program structure. To predict a move in a game, it benefits from tracking the position. Across many examples, learning reusable structure can be more effective than treating every continuation as unrelated.

**A pretrained model is therefore better understood as a learned starting point for solving many tasks than as a collection of stored answers.** This is why it can apply familiar concepts to a new problem and adapt its behavior from examples in the prompt.

We can observe this in controlled settings. A model trained to predict Othello moves learned an internal representation of the board; interventions on that representation changed its move predictions. Separately, transformers trained across regression tasks using next-value prediction learned to predict new functions from examples while keeping their weights fixed at evaluation. The first demonstrates learned state tracking; the second demonstrates a reusable procedure for learning from context. These controlled demonstrations show that prediction training can produce useful internal machinery. [Learned state](https://arxiv.org/html/2210.13382v2), [Learning from examples](https://arxiv.org/html/2208.01066v3)

Increasing pretraining can improve three things: the range of knowledge available, the quality of the computations used to interpret it, and the probability of proposing a useful approach to a new task. That last improvement matters even when the final answer is still wrong: a better partial strategy can be a much better starting point for practice and feedback.

The resource must be allocated well. More parameters provide additional representational capacity; more useful data supplies additional learning opportunities. Chinchilla's controlled scaling experiments showed the value of balancing the two. DataComp-LM demonstrated that changing which data is used can materially change performance under standardized training. “Bigger pretraining” should mean a better-trained starting model, not simply a larger parameter count. [Chinchilla](https://arxiv.org/html/2203.15556v1), [DataComp-LM](https://arxiv.org/html/2406.11794v1)

## 3. Reasoning makes computation available; RL can teach the model how to use it

A direct answer has limited opportunity to work through intermediate states before producing its first answer token. A generated reasoning sequence creates additional steps: each token is computed using the input and the earlier tokens. Intermediate results can be written down, revisited, and used in later decisions. Tools extend this further by providing execution, search, and observations.

**A useful reasoning trace is part of the computation that produces the answer.** This explains why adding intermediate steps can solve problems that are difficult to answer directly. Controlled scratchpad experiments demonstrated this on arithmetic and program execution; separate theory and experiments show how intermediate tokens enable serial computations in shallow transformers. These results establish a mechanism, without requiring every visible explanation to faithfully describe the model's internal process. [Scratchpads](https://arxiv.org/html/2112.00114v1), [Serial computation](https://arxiv.org/html/2402.12875v1)

But more steps help only if they do useful work. Repeating a mistaken premise for another thousand tokens wastes computation. A reasoning model needs a policy—a learned rule for choosing the next thought or action—that uses its budget productively.

RL supplies one way to learn that policy. The model attempts tasks, receives scores, and changes the weights that selected its tokens and actions. A final score does not identify every useful step; across varied attempts, learning favors choices associated with better outcomes. With repeated practice, it can become better at choosing a promising approach, checking an intermediate result, switching strategy, or deciding when further work is unnecessary. The reward can judge the final outcome, intermediate behavior, or both.

For the coding agent, an expert might initially demonstrate “reproduce the bug, inspect the failing path, patch, then test.” RL then lets the agent practice variations and learn which decisions actually lead to a working patch. In this example, the demonstration supplies a useful procedure; experience improves its execution. Neither requires humans to write the exact successful trajectory for every new bug.

There is controlled evidence that **organizing familiar skills is itself learnable**. In string-transformation experiments, after supervised preparation, RL practice on compositions improved performance on deeper combinations that were not trained directly, whereas practicing individual operations was less effective. Knowing how to reverse a string and how to delete a character is different from reliably applying them in the requested order and carrying the intermediate result forward. This gives us a concrete mechanism for capability gains without requiring a new fact or atomic skill on every training example. [Reusable composition](https://arxiv.org/html/2606.18089v1)

### Why “RL just upweights successful answers” is incomplete

It describes part of the update but misses what is shared between attempts. The model is not merely maintaining a separate probability for every complete answer. Shared parameters control decisions across many tasks. Learning a useful decision on one problem can change behavior on other problems; new attempts then come from that changed model.

Some gains are indeed better selection from an existing repertoire. Turning an occasional expensive success into a reliable cheap success is valuable in its own right. Other gains can be better procedures, improved combinations of existing skills, or learning from information supplied by an environment or teacher. The practical distinction is whether the model becomes better at new problems under a fixed inference budget.

Pass@K—the chance that at least one of K attempts succeeds—helps diagnose this, but is not a permanent capability ceiling. Some experiments find that RL improves single-attempt accuracy while narrowing high-K coverage; others find expanded coverage and transfer after different or longer training. This says that learning design changes the outcome. It does not support treating a finite sample of the starting model as the complete set of things it can ever learn. [Coverage contraction](https://arxiv.org/html/2504.13837v3), [Prolonged RL](https://arxiv.org/html/2505.24864v1)

## 4. Pretraining, RL training, and inference compute reinforce one another

**A stronger starting model can make practice more productive, and better practice can make additional inference computation more productive.** This is the key relationship for thinking about further scaling.

Suppose a model has a 0.1% chance of finding a successful solution to a particular problem in one independent attempt. It takes about 1,000 attempts to have a 63% chance of seeing one success. At 1% per attempt, roughly 100 attempts achieve the same chance. This is an illustrative calculation, not a measured scaling law: a modest absolute increase in starting competence can substantially change the cost of obtaining useful experience.

This complementarity also has direct experimental support: controlled chess experiments and a sweep of 14 pretraining checkpoints of one 1B text model found that stronger pretraining improved both post-RL performance at a given RL compute budget and the local rate of improvement during RL. The result supports the mechanism that preparation can make practice more productive. The exact compute allocation remains specific to the task and training regime. [Pretraining–RL scaling](https://arxiv.org/html/2607.16097v1)

The three budgets therefore buy different things. Pretraining improves what the model brings to the problem. RL improves the procedure it has learned through practice. Inference compute lets it apply that procedure more thoroughly to this particular instance. If the procedure is poor or important information is missing, simply increasing inference tokens can have little value. Experiments on adaptive test-time computation find precisely this dependence on task difficulty and the starting model. [Test-time compute](https://arxiv.org/html/2408.03314v1)

From this model, we should expect the following:

**More pretraining should broaden what is cheaply learnable and solvable**, especially where knowledge, representations, or initial problem-solving competence are the bottleneck. Its value depends on useful data and efficient allocation, not size alone.

**More RL on a fixed task distribution should improve useful behavior and then encounter diminishing returns.** Continuing progress will often require harder or more diverse tasks, better exploration, or more informative feedback. Increasing RL compute and improving the practice environment are different interventions; both matter.

**More inference compute should help when additional thinking, attempts, or tool calls have a reasonable chance of producing useful progress.** Training can improve that chance. More attempts become more valuable when we can distinguish good solutions from convincing failures; with a weak evaluator, additional search can instead find better ways to exploit its mistakes. The optimum depends on the task, latency requirements, and the ability to recognize a good result.

Architecture and systems efficiency affect all three budgets. If reading long histories or generating rollouts becomes cheaper, more practice or deeper attempts fit within the same resources. DeepSeek-V4.1's context-handling improvements are an example of this economic mechanism. Kimi K3's combination of changes to model structure, data, and training is another example of improving how effectively compute is used. [DeepSeek-V4.1](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/main/DeepSeek_V41_Tech_Report.pdf), [Kimi K3](https://arxiv.org/html/2607.24653v2)

## 5. Expert data matters because learning needs useful information about what to do and what counts as success

**The scarce ingredient is often a good learning signal.** A model can produce enormous numbers of attempts. Those attempts are valuable when we can identify meaningful successes, diagnose failures, or provide a better approach.

Expert input has several distinct uses. A demonstration shows how to begin a task that the model cannot yet solve. A correction reveals a distinction it is missing. A judgment separates plausible answers from good ones. A test or simulator lets it practice repeatedly. A realistic task distribution tells it what deserves practice in the first place.

These uses address different bottlenecks. If every attempt fails, more binary failure labels may add little; an example, hint, or easier subtask can make progress reachable. Curriculum experiments demonstrate this by initially asking a model to finish a partially supplied solution, then gradually asking it to do more itself. [Reverse curriculum](https://arxiv.org/html/2402.05808v2)

If many attempts look good but contain subtle errors, expert verification can be more valuable than another demonstration. Human step-level judgments have trained better selectors of mathematical solutions. Progress-based feedback has also improved RL by distinguishing useful intermediate steps, including steps in attempts that ultimately fail. A better learning signal can therefore be more valuable than simply generating more attempts. [Process supervision](https://arxiv.org/html/2305.20050v1), [Rewarding progress](https://arxiv.org/html/2410.08146v1)

This leads to a strong practical expectation: **RL will scale most readily where useful outcomes can be evaluated cheaply, repeatedly, and reliably.** Code execution and formal checks often provide this advantage. Open-ended business decisions and scientific research may require costly or delayed judgments. Model-based graders extend what can be scored, but their mistakes become part of the training signal. Improving the evaluator is itself part of improving the learner.

Synthetic data fits naturally here. A strong model can generate examples; a search process can discover solutions; a program can verify them; experts can design and audit the process. Human effort can then supervise many more learning events. But simply generating more text supplies no guarantee that the events are informative.

There are two sources of improvement to distinguish. Reasoning can work out consequences of information the model already has. Interaction can supply new empirical information—a test failure, a measurement, or an unseen company fact. A model can become better through either route. It cannot reliably infer an arbitrary hidden fact that none of its inputs or training reveal.

## 6. Prompting and custom training can implement similar behavior at different costs

**Prompting changes the computation performed using fixed weights. Training changes the reusable weights that perform that computation.** Both can make a model follow a procedure, use examples, or apply a domain-specific rule. This is the real overlap behind the question of equivalence.

Imagine that a general model becomes an excellent bug triager when given a long guide, historical examples, and several attempts to check its work. A specialist can be trained to reproduce that system's useful behavior with a shorter prompt or less search. This is amortization: paying an upfront learning cost to avoid repeating some work on each request.

Context distillation explicitly trains toward this relationship:

> Student answering a question ≈ teacher answering that question with extra instructions, examples, or documents.

The approximation is learned across a distribution of tasks. Context-distillation experiments demonstrate that useful prompt-assisted behavior can be transferred into weights, including with teacher feedback on the student's own attempts. [Context distillation](https://arxiv.org/html/2209.15189v1), [On-policy context distillation](https://arxiv.org/html/2602.12275v1)

**The equivalence is about performance on a workload, not identity of the two systems.** The student may lack capacity, miss rare cases, or require new training when the task changes. A sufficiently capable general model may handle those changes from a new prompt. Conversely, a compact specialist may perform a repeated procedure more cheaply and consistently.

Fresh information creates a sharper limit. If the same question needs different answers depending on today's repository or company policy, a student that cannot see that current state cannot always reproduce a teacher that can. Distilling yesterday's documents does not eliminate today's information requirement. A trained specialist may still need retrieval and tools.

Nor must reusable work live in weights. A tested script, a better prompt, a retrieved playbook, or persistent task memory can also avoid rediscovery. Experiments that generate and reuse task-level reasoning plans illustrate this with frozen models. The engineering choice is where to store what has been learned, how reliably the model uses it, and how expensive it is to update. [Reusable reasoning plans](https://arxiv.org/html/2402.03620v1)

## 7. A company builds an advantage by capturing the missing knowledge and feedback in its workflow

For an existing business workflow, the default route is to start with a capable general model, provide the right information and tools, and learn from its failures. Training a foundation model from scratch discards an enormous amount of reusable capability unless there is a specific reason to rebuild it.

A software company can give a model its repository, development conventions, issue history, and test environment. It then observes where the system fails:

- Missing or stale facts point toward retrieval and tool access.
- Repeated mistakes in a stable procedure point toward better demonstrations, SFT, or distillation.
- Difficulty choosing actions despite having the right information points toward practice with credible outcome feedback, potentially RL.
- Good results at excessive cost point toward distillation, routing, shorter context, or a more efficient model.

**A company-specific model can beat a stronger general model on the company's workload without being generally more intelligent.** It may have better information, better practice on the relevant distribution, or a cheaper learned procedure. To understand the cause, compare systems with the same starting model, information, and inference budget. To choose a deployment, compare their achieved quality, latency, maintenance burden, and total cost.

The most valuable company asset may therefore be the combination of proprietary cases, tools, expert judgment, and outcome feedback. That asset can improve prompting today and train a specialist later. The weights are one way to turn the asset into useful behavior.

The relevant economic volume is the number of **similar tasks before the requirements change**. Training pays when savings across those tasks exceed the cost of data creation, training, maintenance, and refreshes at the required quality. Compare against the best reusable prompt, cached computation, workflow code, or model routing—not a deliberately expensive baseline. High traffic alone is insufficient if requests share little reusable knowledge or procedure, or the baked-in rules soon become stale.

## 8. Automated research closes the loop between trying ideas and retaining what worked

**Automated research is a search-and-learning loop with an experimental evaluator.** A model proposes a change, runs an experiment, interprets the result, and chooses the next change. The proposal model contributes prior knowledge and research procedures; the experiment contributes evidence about this particular idea.

Improvement can persist in three places. The current conversation can guide the next attempt. External memory or code can preserve useful discoveries across attempts. Training can consolidate recurring successes into the proposing model's weights. These are different forms of accumulation, and a research system can improve through the first two without any weight update.

AlphaEvolve demonstrates program proposal, automated evaluation, and reuse of promising candidates in an evolutionary search. Karpathy's autoresearch exposes a compact training-experiment loop: edit code, run a bounded experiment, check the metric, keep or discard the change. In the latter, the model being trained in an experiment is distinct from the agent deciding which experiment to run. [AlphaEvolve](https://arxiv.org/html/2506.13131v1), [autoresearch](https://github.com/karpathy/autoresearch)

The connection to frontier training is direct: useful research results can improve datasets, evaluators, training algorithms, or infrastructure; validated research trajectories can become demonstrations or distillation material; a research environment can supply RL feedback. A better model can then propose better experiments. This is a plausible compounding mechanism, and parts of it are already demonstrated.

The bottleneck moves to **how quickly the system can obtain trustworthy evidence that an improvement transfers**. A faster score on one proxy is not enough if the change fails on a larger model or a new workload. Research with cheap executable tests is easier to automate than research whose decisive experiments take months. A larger model or more agents cannot remove those experimental constraints, although they can help design better tests.

## What this model predicts

The resulting expectation is continued progress from the combination of stronger starting models, richer practice environments, better use of inference computation, and cheaper transfer of successful behavior. The most consequential question about a new system is which of those improved and which bottleneck it removed.

This suggests specialization to grow where tasks repeat and feedback is valuable, while general models remain useful for new tasks and as teachers. This suggests expert work to shift partly toward difficult examples, judgments, and environment design. This suggests automated research to advance fastest where experiments are cheap and improvements can be checked independently.

The main unresolved issue is the breadth and rate of transfer: how much practice in available environments improves performance on genuinely unfamiliar work. We have evidence that transfer occurs and evidence that optimization can become narrow. A general forecast of frontier growth requires knowing which regime future training occupies. The right test is sustained improvement on new tasks at a measured resource budget.
