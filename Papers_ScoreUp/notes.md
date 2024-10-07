# Building an intelligent recommendation system for personalized test scheduling

## Problem Statement
   - **Challenges in Test Scheduling:** The rigid nature of test scheduling does not account for the individual learning pace of students. Over-testing can demotivate students, while under-testing can fail to capture adequate progress.
   - The main objective of the recommendation system was to select the minimal number of critical test administration points for each student during the school year.

## Task Description
 - The main goal of the recommendation system is to **minimize the number of critical test administration points** for each student throughout the school year while still accurately measuring their learning progress. The system uses information from students' past and current test performances to recommend an **optimal sequence of future test timings**. 

 - To avoid excessive testing, the system pre-sets a **limit on the number of tests** a student can take. Within this limit, the system finds the most effective times to administer tests that reflect the student's learning progress accurately. The selected test points should:
   1. Represent the student's **final performance score** at the end of the year.
   2. Accurately track the student's **learning growth** (or progress slope).

 - The system’s objective is to provide the best diagnostic decisions by maximizing the **positive changes in student scores**. It continuously monitors performance changes, ensuring appropriate **interventions** or **feedback** are provided. 
   - For students with steady progress, the system may schedule tests only a few times (beginning, middle, and end of the year).
   - For students with fluctuating scores, more frequent monitoring is required to ensure they receive adequate support.

 **=> Ultimately, the system aims to **reduce testing** to maximize instructional time while ensuring the tests capture students' growth and learning outcomes effectively.**

  #### Example Response from the System:

  **Student ID: 12345**  
  **Current Performance Level:** Moderate Progress (Fluctuating Scores)

  **Recommended Test Schedule:**
   1. **Test 1:**  
    - **Date:** October 5, 2024  
    - **Reason:** Establish baseline performance for early learning progress.

   2. **Test 2:**  
    - **Date:** November 20, 2024  
    - **Reason:** Mid-semester checkpoint to monitor learning and provide early intervention if needed. Recent quizzes suggest inconsistent performance, so an earlier test is recommended.

   3. **Test 3:**  
    - **Date:** January 15, 2025  
    - **Reason:** Post-winter break assessment to check retention of material and assess if further review is necessary.

   4. **Test 4:**  
    - **Date:** March 30, 2025  
    - **Reason:** Near end-of-semester test to gauge final performance and learning outcomes. This test will help determine the final learning progress for the academic year.


## Methodology
   - **Reinforcement Learning Framework**: The system uses RL to select optimal testing points for students, aiming to reduce the number of test administrations while maximizing score improvement.
   - **Markov Decision Process (MDP)**:
     - This is a mathematical framework for modeling decision-making situations where the outcomes are partly random and partly under the ocontrol of a decision maker (agent). They are used in situations where the system is fully observable.
     - In the context of test scheduling, the agent would ideally know the exact state of a student's learning progress. However, **student performance is not fully observable** as you don't have complete knowledge about every aspect of a student's understanding at every moment
    `However, it is important to note that the classifier will not be able to capitalize on the full history of students’ test performance results due to the interactive nature of the computerized formative assessments.`
        => Therefore, **POMDP: Partially Observable MDP** was used
   - **POMDP: Partially Observable MDP**: 
     - The key difference in a POMPD is that the agent (system) does not have complete knowledge of the state. It could only infer the student's true knowledge state through their performance on quizzes, but it can't directly observe everything about their understanding
     - **Belief State**: 
       - Since the agent cannot observe the true state, it keeps a belief about the current state, which is a probability distribution, which is when the agent assigns a certain probability to each possible state, representing how likely it thinks the system (or student) is in that state.
        - The belief state is updated every time the agent receives new information (eg: after a quiz)
        - Eg: Let’s the system is trying to assess how well a student understands a topic. It can’t directly observe the student’s true understanding (state), but it has three possible guesses (states):
        
            1.	State 1 (High Understanding): The student understands the topic very well.
            2.	State 2 (Medium Understanding): The student has a moderate understanding of the topic.
            3.	State 3 (Low Understanding): The student has a poor understanding of the topic.

        Now, based on the student’s past quiz performance, the system assigns probabilities to each state. This is the belief state:

            •	State 1 (High Understanding): 0.5 (50% probability)
            •	State 2 (Medium Understanding): 0.3 (30% probability)
            •	State 3 (Low Understanding): 0.2 (20% probability)

        This means that the system believes:

            •	There is a 50% chance the student has a high understanding.
            •	There is a 30% chance the student has a medium understanding.
            •	There is a 20% chance the student has a low understanding.

        This distribution will change as the system gathers more information, such as the results of the next quiz.

        **=> The system doesn’t know exactly how well the student understands the material (true state), but it maintains a belief state (probability distribution) that reflects its current understanding of the student’s knowledge level. The system updates this belief as it receives new data, like quiz results, allowing it to make more informed decisions about when and how to test the student again.**
  
     => The **goal** is to maximize the **expected cumulative reward** by making decisions based on the current **belief state** \( b(s) \), which is updated as new observations \(o\) are made. This leads to more informed and tailored decisions over time.

<!-- \[
        V(b) = \max_a \left( \sum_s b(s) \sum_{s'} T(s'|s, a) \left( R(s, a) + \gamma V(b') \right) \right)
\]

Where:
      1. \(V(b)\) is the value of the belief state \(b\).
      2. \(\gamma\) is a discount factor for future rewards.
      3. \(b'(s)\) is the updated belief state after taking action \(a\) and observing \(o\).  -->

 - **Task Formalization with POMPD**:
  <!-- - In this system, each **episode** represents a single **test administration** (a moment when a student takes a test). The system doesn't rely on the student's full performance history over the school year to make decisions but instead works **dynamically** by evaluating the student's performance as the tests are taken over time. 
    - **Episode**: An episode happens each time a student takes a test. For example, if a student takes a test in **October**, that would be **episode 1**. If they take another in **December**, that would be **episode 2**, and so on.
      
    - **Test History**: In each episode, the student's **previous and current test scores** are collected into a **test history**. This is a collection of all the scores the student has received so far. 
      - Example: After three test episodes, a student’s test history might look like this:  
        - \( S_1 \) (score from episode 1),  
        - \( S_2 \) (score from episode 2),  
        - \( S_1, S_2 \) (combined results from both episodes).  
        - The history shows how the student’s scores have changed over time.

    - **Mapping History to Success**: The system uses this test history to predict whether the student is making enough progress to succeed at the end of the year. The system needs to figure out, based on past performance, whether the student will achieve the desired learning goals.

    - **Using Machine Learning**: The system can use advanced algorithms like **Recurrent Neural Networks (RNNs)** or **Long Short-Term Memory Networks (LSTMs)** to take the sequence of test scores and predict whether the student is on track for success. These models work well with sequences of data (like test scores over time).

        - **Example of "Episodes"**: Imagine a student takes three tests during the year:

          - **Episode 1** (Test in October): The student scores **70%**. The system stores this result.
          - **Episode 2** (Test in December): The student scores **75%**. Now, the system has a sequence: **70% (episode 1)**, **75% (episode 2)**.
          - **Episode 3** (Test in March): The student scores **80%**. The system now has the full history: **70%, 75%, 80%**.

    **=> The system uses this history to predict if the student will succeed by the end of the year. If the student's progress is steady, the system might decide no further tests are needed. If the scores fluctuate, the system might schedule more tests to track improvement.**

    - **Why This Matters:**
    By breaking the year into episodes (test moments) and using the **history of test scores**, the system can dynamically schedule tests and make predictions about the student’s success without needing all test data at once. -->

  - Within the **POMDP framework**, the system essentially performs two main tasks:
      - **Forming the Episode**:
        - Each **episode** represents a test administration (a test moment).
        - The system tracks the student’s test score and test history within each episode. It uses this sequence of episodes to build a **learning trajectory** of the student’s performance over time.
        - The episode provides the data needed to evaluate the student's progress at key moments without requiring the system to have the full test history at every point. Instead, the system uses the most recent test results to make dynamic decisions.

      - **Guessing the Next State (Belief State)**:
        - Since the system cannot directly observe the student’s true level of understanding, it uses a **belief state** to estimate or "guess" the student's current learning state.
        - This belief state is updated after each test (episode), using the student's latest test score and previous performance history.
        - To refine this belief, the system uses **Gated Recurrent Units (GRUs)**, which help process and interpret the sequences of test scores to predict the student's learning progression.
        - Based on this updated belief, the system decides when to schedule the next test (the next episode) to optimize the learning process.

    **=> In summary**
        - **Task 1 (Episode Formation)**: Track test results over time and structure the year into test moments (episodes).
        - **Task 2 (Guessing the Next State)**: Continuously update the belief state about the student's knowledge using GRUs, based on past and current performance, to decide when the next test should be administered.

    - **Actor-Critic Algorithm**: The system combines an actor and critic model to optimize decision-making in selecting test points.

- **System development using Actor-Critic algorihtms**
    1. **Baseline \( V \)**:
       - The **baseline** \( V \) (also known as the value function) represents the **average expected reward** the system can expect from being in a particular state \( s_t \). It helps in stabilizing the learning process by comparing the actual outcome of an action to this expected outcome.
       - **Example**: If a student has done well in past quizzes, the baseline might represent the system's expected score for the next quiz based on their previous performance.

    2. **Value \( Q \)**:
       - The **value function \( Q \)** is the expected reward from taking a specific **action** \( a_t \) in a specific **state** \( s_t \). It tells the system how valuable it is to take a particular action in a particular situation.
       - **Example**: If the system schedules a test, the **value \( Q \)** could represent the expected improvement in the student's score from taking that test.

    3. **Terminal Episode \( T \)**:
       - The **terminal episode** is the final step or end point in an episode (sequence of actions). In this case, it represents the **last test administration** in the system. When the terminal episode is reached, the system evaluates how successful its decisions were, based on the final outcomes (e.g., the student’s final performance).
       - **Example**: If the school year ends with a final test, that would be the terminal episode \( T \), where the system assesses the student's learning progress.

    4. **Advantage \( A \)**:
       - The **advantage function \( A \)** measures how much better (or worse) a specific action was compared to the baseline. It's calculated as the difference between the **actual reward \( Q \)** for an action and the **expected reward \( V \)** (baseline).
       - **Example**: If scheduling a test resulted in a much higher student score than expected, the advantage would be positive, indicating that scheduling the test was a good decision.

    5. **Discounting Value**:
       - The **discounting value** (often represented as \( \gamma \)) is used to give more importance to immediate rewards and less importance to future rewards. A discount value close to 1 (like \( \gamma = 0.99 \)) ensures that the system considers both immediate and future rewards almost equally.
       - **Example**: The system might prefer scheduling tests early but still takes into account the long-term impact on the student's overall performance throughout the year.

    **Summary of the System's Actor-Critic Algorithm:**
       - The system uses the **actor-critic algorithm** to make decisions about test scheduling. The **actor** (decision-maker) selects actions (like when to schedule a test), while the **critic** evaluates the outcomes by comparing the reward from the selected action (\( Q \)) to the expected reward (\( V \)).
       - The system aims to **maximize rewards** (student performance improvements) by minimizing the number of tests while still providing valuable learning data.
       - The system updates its decisions using the **policy gradient**, adjusting its strategy based on whether the scheduled tests were effective.
       - By experimenting with the **terminal episode \( T \)** (final test points) and using a **discounting value** close to 1, the system ensures it considers long-term student performance while still keeping the number of test administrations low. 

## How everything ties together

1. **Reinforcement Learning (RL)**:  
   - **What it is**: RL is the **overall methodology** for training the system. It allows the system (called the **agent**) to learn by interacting with the environment and receiving feedback (rewards). The agent aims to **maximize cumulative rewards** by making the best decisions over time.
   - **In the system**: The goal of RL here is to optimize **test scheduling** for students. The system learns how to decide the **best times to schedule tests** by trial and error, using rewards based on student performance improvements.
   - **Mathematical Representation**: The system seeks to maximize the expected cumulative reward, which is the sum of rewards \( R \) over time:
     \[
     J(\Theta) = E \left[ \sum_{t=1}^{T} r_t \right]
     \]
     where \( r_t \) is the reward at time step \( t \), and \( T \) is the terminal episode (last test point).

2. **Actor-Critic Algorithm**:  
   - **What it is**: The **Actor-Critic** algorithm is an implementation of RL that **makes the actual decisions** about which action (e.g., scheduling a test) to take. It consists of two parts:
     - **Actor**: Responsible for selecting actions based on a **policy** \( \pi(a|s) \), which gives the probability of taking action \( a \) given the state \( s \).
     - **Critic**: Evaluates the action by calculating a **value function** \( V(s) \), which tells the agent how good or bad the chosen action was, and **advantage function** \( A(s, a) \) to compare the action to a baseline.
   - **In the system**: The Actor decides when to administer tests, while the Critic evaluates how effective each test was based on student performance.
   - **Mathematical Representation**:
     - The **policy gradient** used to update the policy (i.e., how decisions are made) is given by:
       \[
       \nabla_\theta J(\Theta) = E \left[ \nabla_\theta \log \pi_\theta (a_t|s_t) A(s_t, a_t) \right]
       \]
       where \( A(s_t, a_t) = Q(s_t, a_t) - V(s_t) \) is the **advantage function** that measures how much better or worse an action is compared to the expected value \( V(s_t) \).
     - The **advantage function** \( A(s_t, a_t) \) is:
       \[
       A(s_t, a_t) = Q(s_t, a_t) - V(s_t)
       \]
     - The Critic estimates both the value \( V(s_t) \) and the action-value \( Q(s_t, a_t) \), and uses the difference to update the Actor’s decisions.

3. **Partially Observable Markov Decision Process (POMDP)**:  
   - **What it is**: The **POMDP** is the **framework** within which the Actor-Critic algorithm operates. It models the system's decision-making under **uncertainty** because the true state of the student’s knowledge is only partially observable.
   - **In the system**: Since the system doesn’t fully know the student’s actual understanding level, the POMDP helps the Actor-Critic algorithm make decisions based on an **estimated belief state** (a probability distribution over possible student states). This belief is updated with new information (test scores) after each test.
   - **Mathematical Representation**:
     - The **belief state** \( b(s) \) is a probability distribution over possible states \( s \), updated after each observation \( o \). The system updates its belief using the observation function \( O(o | s', a) \), which gives the probability of observing \( o \) after transitioning to state \( s' \) by taking action \( a \).
     - The **POMDP objective** is to maximize the **expected cumulative reward**:
       \[
       V(b) = \max_a \left( \sum_s b(s) \sum_{s'} T(s'|s, a) \left( R(s, a) + \gamma V(b') \right) \right)
       \]
       where \( V(b) \) is the value of the belief state, and \( \gamma \) is the discount factor.

    **=> Overall Picture:**

    1. **Reinforcement Learning (RL)**: The system is built using RL, where the **goal** is to maximize the student’s learning progress (reward) by scheduling tests at optimal times.

    2. **POMDP Framework**: Since the system can’t directly observe the student’s exact level of understanding (the true state is **partially observable**), the POMDP framework helps the system **estimate** the student’s current learning state based on test performance. It updates this **belief state** after each test.

    3. **Actor-Critic Algorithm**:
    - The **Actor** makes decisions on **when to schedule the next test** by selecting actions based on the current belief state.
    - The **Critic** evaluates the effectiveness of the test scheduling by comparing the actual reward (student performance improvement) to the expected reward (baseline).
    - The **advantage function** helps refine this decision-making process by telling the system how much better or worse the current decision (test schedule) is compared to previous decisions.

    4. **Feedback Loop**:
    - After each test (episode), the system receives feedback (reward based on the student's performance), updates its belief about the student’s knowledge, and adjusts its policy for scheduling future tests. The goal is to **maximize the cumulative reward** while **minimizing the number of tests**, ensuring that tests are only administered when they provide meaningful insights into the student's learning progress.

   **=> Conclusion:**
    - **Reinforcement Learning** is the overall methodology that allows the system to learn how to schedule tests optimally.
    - **Actor-Critic** is the RL **algorithm** that makes decisions about test scheduling, with the Actor deciding when to test and the Critic evaluating the decision.
    - **POMDP** is the **framework** that helps the system deal with the **uncertainty** of not knowing the student's true state of understanding. It uses the belief state and observations (test scores) to guide the Actor-Critic algorithm’s decision-making.

## Results and Evaluation
   - **Classification Accuracy**: The system achieved high accuracy (~89%) and an F1-score (~0.93) in classifying whether students were at risk or exemplary, while reducing the number of test administrations.
   - **Optimal Test Windows**: The study found that **3 test administrations** provide the highest balance between accuracy and diagnostic value. Increasing the number of test windows beyond this threshold did not yield significantly better results.
   - **Score Improvement**: The system was able to maximize the positive score change between test administrations. Fewer test administrations resulted in higher average score improvements per test.
   
## Implications and Benefits
   - **Reducing Over-Testing**: The system successfully reduced unnecessary test administrations without compromising the ability to track student progress.
   - **Critical Test Points**: The RL system identified “critical points” where tests should be administered to capture meaningful performance changes, preventing excessive testing.
____

## Applications

### 1. **Optimization of Quiz Timing**
   - **Test Administration Optimization**: The study’s focus on minimizing unnecessary test administrations can be adapted to optimize **when** quizzes are administered. This means you could design the system to provide quizzes only when the student is likely to benefit the most, based on their learning progress.
   - **Action**: Implement an RL algorithm similar to the one in this study to dynamically schedule quizzes based on a student's previous performance, reducing over-quizzing and preventing fatigue.