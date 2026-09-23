# How companies should think about internal AI

*Private expertise, inference budgets, and learning from experiments*

A company can benefit from better general models even when none of its proprietary information appeared in their training. The model brings reusable capabilities for learning, reasoning, and acting. The company supplies the private facts, tools, data, and experiments on which those capabilities operate.

Custom training offers a different possibility: accumulated private experience can change how a system approaches future work. That may make familiar tasks cheaper, but it can also improve which hypotheses are proposed, which experiments are run, and which apparent successes are rejected. The stronger claim requires evidence that the learned behavior transfers to new problems.

The practical question is how to combine these resources. When should a company improve context and tools, spend more inference compute, preserve experience in external memory, or adapt a model's weights?

## A running example: research inside a company

Consider a hypothetical researcher at a company developing a proprietary industrial process. The company owns process designs and simulation code, maintains experimental equipment, and has collected internal measurements across many trials. Some knowledge is documented. Some lives in failed experiments, equipment quirks, and researchers' experience of which results survive replication.

The researcher wants an AI assistant to investigate a promising change to the process. The assistant can inspect code and records, analyze measurements, propose hypotheses, and run simulations. Physical experiments consume equipment time and materials, so proposals must be selected carefully and run through the company's experimental workflow.

A candidate appears to improve yield in simulation. Before recommending a larger trial, the assistant needs to ask whether the gain depends on a calibration error, a changed measurement procedure, an unrealistic simulator assumption, or a real mechanism. It must choose evidence that distinguishes these explanations.

This example separates three kinds of resources: proprietary intellectual property, observations collected through experiments, and the setup that makes new observations possible. Access to one does not substitute for the others. Reading a process description cannot establish whether a new modification works; running a simulation cannot establish whether its assumptions hold in the physical process.

## 1. Why general models can improve on private tasks

### Private details often have general structure

The assistant must learn an unfamiliar data schema, understand the simulator's interface, track experimental conditions, write analysis code, and interpret results. The names and conventions are proprietary. Program comprehension, statistics, debugging, and experimental design have broader structure.

Training elsewhere can improve these reusable abilities. A stronger model may infer more from the same examples, notice an inconsistent assumption, or design a test that reveals undocumented behavior. Controlled transformers trained across regression tasks demonstrate one part of this principle: they learn to infer new functions from examples while their weights remain fixed during evaluation. That is a controlled demonstration of learning from context, not a complete account of commercial model behavior. [Learning new functions from examples](https://arxiv.org/html/2208.01066v3)

**The absence of company data from pretraining means that private information must enter through another channel. It does not mean private training is necessary before a model can use that information.**

There is a boundary. If two internal implementations expose the same visible information but require different answers, additional reasoning cannot reliably recover the hidden distinction. The assistant needs a file, example, measurement, or expert correction. An undocumented fact may still be discoverable through source inspection or an experiment, but it must become observable somehow.

### Domain judgment can also transfer

Suppose a process modification improves average simulated yield but performs poorly in physical trials. A useful hypothesis is that the trial conditions differ systematically from the simulated conditions. Another is that the measurement changed alongside the process.

Recognizing those possibilities draws on general ideas about selection, measurement, and causal inference. It does not require memorizing the company's process. Private evidence is still needed to determine whether either explanation is correct.

A weaker model may know each component yet fail to combine them. Better representations, reasoning, and practice can make the relevant connection more likely. An expert may recognize a pattern immediately; a model may reach the same hypothesis through a longer derivation. Either can contribute if the hypothesis leads to a discriminating test.

This matters for the investment case. Calling expertise "tacit" does not establish that it must be taught through weight updates. Companies should ask which judgments a strong general model can reconstruct from the available evidence, and which require experience that remains expensive to transmit or rediscover.

## 2. Diagnose what the assistant is missing

| What is missing? | Research example | How to supply or learn it |
| --- | --- | --- |
| Current facts | Equipment calibration, dataset version, simulator interface | Versioned records, retrieval, and tools |
| Recurring procedures | How to prepare a trial, validate measurements, and record a result | Worked examples and executable workflows; training may improve recurring behavior |
| Experienced judgment | Which explanation to test first; when an apparent gain is suspicious | Decision cases, failed experiments, expert corrections, and practice with feedback |
| Unknown empirical relationships | Whether a process modification improves yield under new conditions | New measurements and experiments |

A repository dump cannot replace the history of why researchers rejected attractive ideas. Historical experience cannot answer a genuinely new empirical question without further evidence.

The distinction also clarifies what should remain outside model weights. Today's calibration and latest dataset version require current information. A recurring habit of checking calibration before launching a costly experiment may be worth teaching as a procedure.

## 3. What context, tools, and more inference compute can buy

### Good context is a compact apprenticeship for the task

For the process investigation, useful context includes the objective, acceptance criteria, relevant designs and data contracts, measurement procedures, a worked analysis, known failure cases, and access to inexpensive checks. The assistant needs to know how evidence will be judged: reproducibility, measurement validity, performance across operating conditions, and realistic resource costs.

"Act as an expert researcher" supplies little of this information. Asking an expert to reconstruct the context for every request can also consume the time the system was intended to save. The engineering work is making relevant knowledge available, current, and usable.

With suitable context and tools, a general model can develop substantial task-specific competence during a session. Tests, scripts, experiment records, and decision notes can preserve useful discoveries across sessions. Weight updates are one way to retain learning; external memory and improved workflows are others.

An archive is not automatically usable experience. Retrieval can miss the relevant case, summaries can omit decisive conditions, and long histories can be expensive to interpret. Training offers a different form of compression: repeated experience can change the decisions a model tends to make before it retrieves a particular case.

### Four different purchases are often called "more tokens"

**More relevant context buys information.** It helps when the task or environment is unfamiliar. Unrelated documents can add confusion rather than value.

**More reasoning buys computation on the available information.** It can support derivation, comparison, implementation, and checking. It cannot reliably recover a missing measurement.

**More attempts buy search.** Different approaches may expose a useful solution. At an independent 20% success probability per attempt, eight attempts have roughly an 83% chance of containing a success. This calculation assumes independence and says nothing about whether the deployed system can recognize the successful attempt.

**More tool interactions buy feedback.** A test result or diagnostic measurement changes what the assistant knows. These interactions also consume wall-clock time, experiment compute, equipment capacity, and sometimes expert attention.

The allocation should depend on the next useful action. A fixed maximum reasoning budget for every task wastes resources on easy cases while potentially undersupplying difficult investigations.

Selection deserves its own evaluation. CodeMonkeys reported a correct patch among its candidates on 69.8% of SWE-bench Verified tasks, while its selection procedure achieved 57.4%. Generating a useful answer somewhere in a collection is a different capability from returning it reliably. [CodeMonkeys](https://arxiv.org/html/2501.14723v1)

### Budget for the completed outcome

Model cost includes all calls, failed attempts, retries, and selectors:

> Model cost = ordinary input × input rate + cache writes × write rate + cache reads × read rate + billable output and reasoning × output rate.

Then add retrieval, tools, simulations, physical experiments, serving, and human review. Use the provider's actual billing rules when estimating a deployment.

A ten-step agent that averages 50,000 input and 3,000 output tokens per step consumes 500,000 input and 30,000 output tokens. At illustrative rates of $3 and $15 per million tokens respectively, that costs $1.95. Eight such attempts cost $15.60 before selection, tools, or review. These are accounting assumptions, not current price quotes or claims that additional attempts improve quality.

Caching may reduce repeated-input costs. Parallelism reduces waiting time without eliminating aggregate resource use. In experimental work, equipment and researcher time may dominate both.

**The economic objective is cost per independently accepted outcome, considered alongside latency and expert effort.** At an illustrative $200 per expert hour, avoiding 15 minutes of review releases $50 of capacity. Saving $1 in model costs while adding that review burden is a poor trade. Released capacity is not automatically a cash saving.

For research, an accepted outcome may be a sound rejection of a hypothesis or a reproducible explanation of a failure. Requiring every investigation to produce a positive discovery encourages selective reporting.

### More search requires stronger validation

The researcher can try thousands of modifications against a simulator and select one that exploits its inaccuracies. The simulation score improves while the physical process does not.

Increasing search therefore increases the importance of independent validation. Reserve unseen operating conditions, separate model selection from final evaluation, test alternative assumptions, and use prospective measurements when appropriate. Retain failed variants as well as successful ones so the eventual result can be interpreted in light of the search that produced it.

Cheap checks should precede costly trials. The limiting resource may be informative experiments rather than generated text. The best assistant may be the one that avoids an uninformative equipment run, not the one that produces the most candidate ideas.

## 4. When custom training is justified

**Custom training is justified when it improves useful performance or full cost on new internal tasks relative to the strongest practical alternative.** The alternative includes a general model with good context, tools, memory, and an appropriate inference budget.

For the researcher, the capability target is a better rule for choosing the next hypothesis, diagnostic, experiment, or stopping decision. This research procedure is distinct from the simulator or predictive model being investigated.

There are three investment cases:

1. **Reliability and cost.** An adapted model performs recurring work with fewer retries, shorter contexts, or less expert repair.
2. **Accumulated expertise.** Training helps the model use lessons distributed across many past cases, improving decisions within realistic context and inference budgets.
3. **Learning through private experiments.** Internal environments and outcomes provide informative experience unavailable to outside models, allowing practice to improve behavior on future tasks.

These cases can coexist. A model that saves money need not demonstrate novel research capability to be useful. A claim of better research judgment, however, requires evaluation beyond familiar workflow completion.

### Capture decisions and evidence, not just final reports

When an apparent yield improvement appears, one assistant might launch a larger parameter sweep. Another might first inspect calibration drift or a changed measurement pipeline. Both may know the same public statistics. They differ in which explanation they consider and which action they choose.

A useful training record preserves the evidence available at the time, candidate actions, expert corrections, experiment results, and the conditions under which the lesson transfers. Comparisons between plausible choices can be particularly informative. A polished final report often hides the decisions that made the investigation productive.

Training changes shared parameters, so lessons from one investigation may alter behavior on others. That transfer is a hypothesis to test, not an automatic consequence of collecting more records.

### Match the method to the failure

**Continued pretraining** may improve familiarity with a large, recurring private corpus. It does not by itself define successful research judgment.

**Supervised fine-tuning** teaches verified demonstrations: use an interface, construct a reproducible analysis, diagnose a failure, or document a justified rejection. Include varied cases and recoveries rather than only clean successes.

**Context distillation** trains a student to reproduce useful behavior from a teacher that sees richer context. The approximation is workload-specific. Distilling yesterday's documentation does not remove the need for today's facts. [Context distillation](https://arxiv.org/html/2209.15189v1)

**Reinforcement learning** lets the model practice actions and learn from feedback. It is attractive when useful outcomes can be evaluated repeatedly and reliably. If every attempt fails, sparse pass/fail feedback may be insufficient; demonstrations, hints, partial tasks, or better exploration can make learning possible.

Physical research adds constraints. Simulations and replay tasks may support inexpensive practice, but improvements must survive independent checks against the real process. Training longer against a weak proxy can strengthen the wrong behavior.

**Training a critic or experiment selector** may be useful when good candidates already occur but are ranked poorly. It cannot repair a proposal system that never generates useful options. A learned evaluator must itself be checked against independent outcomes and expert decisions before its scores become a training reward.

## 5. Research progress can persist without weight updates

An assistant can produce a useful result during an investigation. That result can improve later work through a script, design, test, or experiment record. Training can additionally consolidate experience into the assistant's behavior. These are three distinct forms of progress.

AlphaEvolve demonstrates discovery through model-generated programs, automated evaluation, and an evolving archive. It supports the possibility of useful search with feedback; it does not establish that every scientific simulator supplies an equally trustworthy evaluator or that adapting the proposal model is necessary for every discovery. [AlphaEvolve](https://arxiv.org/html/2506.13131v1)

A company's durable assets include well-defined problems, informative experimental environments, failed and successful investigations, expert corrections, and independent evaluation. A model checkpoint is one possible product of that learning process.

Preserving experience must leave room for novelty. Overtraining on historical successes can narrow proposals and institutionalize assumptions that no longer hold. The assistant should retain the conditions under which a lesson applied and seek evidence that could overturn it.

The assistant also need not consume every raw sensor reading as text or operate on the equipment's real-time control path. It can write analysis code, inspect summaries, invoke numerical models, and design experiments. Improving the research process is a separate objective from replacing a production predictor or controller.

## 6. The evidence that should decide the investment

Compare the current workflow, a strong general-model system with appropriate context and tools, that system with more adaptive inference, and an adapted model with the same permitted access. Retain the unadapted starting checkpoint to identify what training contributed. Compare against the strongest practical system to establish business value.

Evaluate at three levels:

| Level | Example | Evidence to collect |
| --- | --- | --- |
| Research engineering | Implement a new analysis using internal libraries | Independent acceptance, serious errors, review time, latency, and full cost |
| Experimental judgment | Diagnose an apparent process improvement | Valid experiments, discrimination between explanations, detection of misleading evidence, and expert intervention |
| Prospective research | Investigate a new process modification | Replicated findings and useful rejections within a fixed experiment and review budget |

Hold out project families and failure types, not only near-duplicate examples. Replay historical cases with only the information available at the time, include abandoned investigations, and protect an untouched prospective evaluation stream. Repeatedly inspecting the final holdout turns it into development data.

Test the evaluator too: seed plausible errors and check whether it rejects persuasive but unsupported conclusions. Where feasible, final reviewers should inspect artifacts without knowing which system produced them.

### Measure the marginal value of expert input

Start from the same model and training problems, then vary whether the examples receive expert corrections. Match training tokens, optimization, runtime information, and inference budgets. Test with and without a shared RL stage, since expert input may matter by making later practice more productive.

A cheaper first experiment compares expert and model-generated casebooks supplied in context. If retrieval captures the gain, expertise is valuable but adapting weights is not yet justified.

The quantity of interest is additional useful performance per expert hour, compared with spending those resources on synthetic examples, better tools, or more inference. Synthetic examples already inherit knowledge from their teacher, so this measures the contribution of additional expert input rather than the total historical contribution of human expertise.

### Account for the useful lifetime of specialization

For a cost-only investment:

> Break-even task volume = fixed adaptation cost / net saving per task.

With illustrative values of $100,000 in fixed costs and $5 saved per task, break-even is 20,000 tasks. Fixed costs include data preparation, expert work, training, evaluation, and deployment. Net savings must include serving, retries, review, and maintenance. If the specialization becomes obsolete before enough similar tasks occur, the cost case fails.

A quality improvement needs its own evidence and value argument. Research findings, avoided experiments, and faster decisions have different consequences; they should not be forced into an unsupported revenue forecast.

General-model progress can replace a specialist, improve its teacher, or provide a better starting checkpoint. Preserving portable datasets, tools, and evaluations makes those upgrades easier to use. Revisit the comparison when the public baseline or internal workload changes materially.

## A practical decision rule

Begin with a consequential research workflow and identify its limiting resource. Supply missing facts through current context and tools. Preserve reusable procedures in tested workflows and accessible records. Spend additional inference where a specific next action is likely to resolve uncertainty. Consider training when recurring experience can improve future behavior enough to justify the cost of learning and maintaining it.

For the hypothetical researcher, success is an assistant that makes the investigation more productive: it implements the analysis correctly, notices when an attractive result is misleading, and chooses a useful next experiment with less expert repair. Whether that improvement comes from context, inference, external memory, or adapted weights should be established through comparison.

General models provide transferable capability. Companies provide private information and opportunities to obtain new evidence. The value comes from turning those resources into better verified work—and retaining what the work teaches.

---

For the underlying learning mechanisms, see [How frontier AI models learn](how-frontier-ai-models-learn.md).
