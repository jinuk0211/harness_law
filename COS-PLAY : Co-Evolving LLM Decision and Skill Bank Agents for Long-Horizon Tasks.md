<img width="984" height="724" alt="image" src="https://github.com/user-attachments/assets/b6ba5a82-06a8-4fbf-bcdc-bdd031f7a4be" />

(a) Decision Agent AD: an LLM-based agent that interacts
with the game through primitive actions and skill retrieval. At each step, it summarizes the
current state, retrieves relevant skill candidates from the skill bank, updates its intention,
selects or switches skills when needed, and executes an action. 

(b) Skill Bank Agent AS: an
LLM-based pipeline that converts unlabeled trajectories into reusable protocol-based skills
and learns compact effect contracts for them. It updates the skill bank by proposing new
skill candidates, refining low-quality skills, and revising skill protocols over time.

<img width="552" height="746" alt="image" src="https://github.com/user-attachments/assets/7f67314c-fef1-45e8-b8e7-5ae17878807f" />
