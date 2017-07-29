# ML-IDS
Applying machine learning principles to the information security field through intelligent intrusion detection systems.

This is a three phases project that can roughly described as

- Data gathering
- Model developpement
- Hardware implementation

## Data gathering

There are a few freely available datasets that applies to the field, I could attempt to create my own set by setting up a virtual machine exposed to the Internet but it is unlikely that I will record any privilege escalation attack (or any sophisticated attacks really) only DoS, DDoS, SSH bruteforce and scans could really be recorded.

### Freely available datasets

- [DARPA Intrusion Detection Data Sets](https://www.ll.mit.edu/ideval/data/) (1998-2000)
- [KDD Cup](https://kdd.ics.uci.edu/databases/kddcup99/kddcup99.html) (1999)

### Not freely available datasets

- [UNB ICX IDS 2012](http://www.unb.ca/cic/research/datasets/ids.html) (Request sent)

For the first part of the prototyping process, the KDD Cup dataset will be used as it was preprocessed already. It uses the following inputs and the goal is to categorize the example in one of the five categories.

### Considerations over the KDD Cup 99 dataset

While one of the few datasets freely available on the subject, the KDD Cup 99 dataset has many shortcomings that can be summarized to:

- Categories are unbalanced, 70% of the testing set is DoS
- Detecting DoS is overall pointless as it generates a log of traffic
- Redundant records

All those points and many more are described in *[A detailed analysis of the KDD Cup 99 Data Set](http://www.ee.ryerson.ca/~bagheri/papers/cisda.pdf)* by Mahbod Tavallaee, Ebrahim Bagheri, Wei Lu, and Ali A. Ghorbani. They proposed a solution which will be partially used namely:

- Removing redundant records
- Artificially increase the number of R2L and U2R examples

### Inputs

For now the project will be based on KDD's data which includes 40 values, 32 continuous, 8 of which are symbolic and will have to binarized if they aren't already. While some value are described as continuous in the dataset, it is important to note that it isn't the mathematical definition in the sense that { 0, 1, 2, 3, 4 } will be described as continuous.

When symbolic variables are binarized, the resulting vector contains 3 (protocol) + 70 (service) + 11 (flag) + 37 (continuous or already binarized) = 121 values

While all of those values will be used in the first version to allow comparison with current research, some might be added afterwards for sake of completeness and relevancy in 2017.

| Variable name               | Discrete or Continuous | Possible values  |
| --------------------------- |:----------------------:| ----------------:|
| Duration                    | Continuous             | [0, ∞[           |
| Protocol Type               | Discrete (Symbolic)    | {icmp, tcp, udp} |
| Service                     | Discrete (Symbolic)    | {IRC, X11, Z39_50, aol, ..., hostnames, http, ftp}      |
| Flag                        | Discrete (Symbolic)    | {OTH, REJ, RSTO, RSTOS0, RSTR, S0, S1, S2, S3, SF, SH}  |  
| Source bytes                | Continuous             | [0, ∞[           |
| Destination bytes           | Continuous             | [0, ∞[           |
| Land                        | Discrete               | {0, 1} |
| Wrong fragment              | Continuous             | [0, 3] |
| Urgent                      | Continuous             | [0, 5] |
| Hot                         | Continuous             | [0, 77] |
| Num failed                  | Continuous             | [0, 5] |
| Logged in                   | Discrete               | {0, 1} |
| Num compromised             | Continuous             | [0, 1] |
| Root shell                  | Continuous             | [0, ∞[ |
| Su attempted                | Discrete               | {0, 1} |
| Num root                    | Continuous             | [0, 2] |
| Num file creations          | Continuous             | [0, 40] |
| Num shells                  | Continuous             | [0, 2] |
| Num access files            | Continuous             | [0, 9] |
| Num outbound cmds           | Continuous             | { 0 } |
| Is host login               | Discrete               | {0, 1} |
| Is guest login              | Discrete               | {0, 1} |
| Count                       | Countinuous            | [0, 511] |
| Srv count                   | Countinuous            | [0, 511] |
| Serror rate                 | Countinuous            | [0, 1] |
| Srv serror rate             | Countinuous            | [0, 1] |
| Rerror rate                 | Countinuous            | [0, 1] |
| Srv rerror rate             | Countinuous            | [0, 1] |
| Same srv rate               | Countinuous            | [0, 1] |
| Diff srv rate               | Countinuous            | [0, 1] |
| Srv diff host rate          | Countinuous            | [0, 1] |
| Dst host count              | Countinuous            | [0, 255] |
| Dst host srv count          | Countinuous            | [0, 255] |
| Dst host same srv rate      | Countinuous            | [0, 1] |
| Dst host diff srv rate      | Countinuous            | [0, 1] |
| Dst host same src port rate | Countinuous            | [0, 1] |
| Dst host serror rate        | Countinuous            | [0, 1] |
| Dst host srv serror rate    | Countinuous            | [0, 1] |
| Dst host rerror rate        | Countinuous            | [0, 1] |
| Dst host srv rerror rate    | Countinuous            | [0, 1] |

### Outputs

There are multiple possible outputs that can be grouped in

| Label       | Description                         | Sub-labels |
| ----------- |:-----------------------------------:| ---------- |
| normal      | Normal                              | { normal } |
| dos         | Denial of Service Attack            | { back, land, neptune, pod, smurf, teardrop } |
| probe       | Probe                               | { ipsweep, nmap, postsweep, satan } |
| u2r         | User to root (Privilege escalation) | { buffer_overflow, loadmodule, perl, rootkit } |
| r2l         | Remote to user                      | { ftp_write, guess_passwd, imap, multihop, phf, spy, warezclient, warezmaster } |

## Model developpement

While researchers already explored most machine model, they forgot one aspect that is crucial to the project at hand: resources. The whole point being to end up with a usable hardware implementation, it is impossible to afford more than a few milliseconds for each packets on a rather weak machine. I will refrain from going straight to the LSTM and take the time to benchmark the performance of other well-known algorithms. I will explore three models to identify the one that shows the best performance to resources among logistic regression, support-vector machines, deep feedforward neural networks, and LSTM recurrent networks.

To avoid reinventing the basics, I will use the TensorFlow framework that supports both x86-64 and ARM.

### Logistic regression

- Pros
    - Fast inference
    - Simple to understand because it is not a blackbox
- Cons
    - Probably too simple to correctly fit the problem
    - No conception of time

### Support Vector Machines
- Pros
    - Usable for semi-supervised learning if using TSVM (unlabelled data)
    - It finds the global minimum
- Cons
    - A normal SVM will not return a prediction confidence and it's a big handicap in later implementation.
    - No conception of time

### Standard FeedForward Neural Network
- Pros
    - Complex enough to fit most problem
- Cons
    - No conception of time

### LSTM Recurrent Neural Network
- Pros
    - Works with sequence, in our case, the packet that came before could be related to the next one. Such relation wouldn't be handled by the above models.
    - Documented use in IDS
- Cons
    - More complex than all three of the above.

## Hardware implementation

### Training

I will use my personnal computer for the training, if someone wishes to replicate the process, AWS offers VPS with GPUs []().

- AMD Zen 1700
- Nvidia GTX 1080Ti

### Inference

These are merely supposition on what minimal hardware would be required based on the platform. In the case at hand I will use an x86-64 processor because the cost for a dual gigabit port is lower and the hardware will be supported out of the box (I'd rather avoid troubleshooting hardware issues).

- x86-64
    - 2 cores celeron processor (would scale if more processing power was available)
- ARM
    - 4 cores ARMv8 or more recent
- 2-4 GB RAM

For portability and performance, the inference will be done in C++.

## References

All consulted papers & documentation is available in the "papers" folder.