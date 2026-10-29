# Low Performance Compute Cluster

This contains setup and maintenance details for my "low performance compute cluster". Consider it more of a lab notebook for now...

I got the idea to build a homelab bioinformatics (or other) server by cobbling together a couple of computers that on their own would be unimpressive. 


```mermaid
flowchart TD
    rpi4["Raspberry Pi 4<br/>Head Node<br/>"]
    comp1["Node 1<br/>Desktop<br/>Cores: 4<br/>Threads: 4<br/>RAM:32GB DDR3<br/>SSD: 1TB<br/>HDD: 500GB<br/>GPU: RTX 2080 Super (8GB)<br/>2nd GPU incoming?"]
    comp2["Node 2<br/>2014 Macbook Air<br/>Cores: 2<br/>Threads: 4<br/>RAM:8GB DDR3<br/>SSD: 256GB<br/>HDD: N/A"]
    comp3["Node 2<br/>2014 Macbook Pro<br/>Cores: 4<br/>Threads: 8<br/>RAM:16GB <br/>SSD: 500GB<br/>HDD: N/A<br/>GPU: GeForce GT 750M (2GB)"]
    
    rpi4 --> comp1
    rpi4 --> comp2
    rpi4 --> comp3
    
    classDef headNode stroke:#818cf8,fill:#eef2ff
    classDef childNode stroke:#4ade80,fill:#f0fdf4
    
    class rpi4 headNode
    class comp1,comp2,comp3 childNode
```

## Thanks
This has been a great learning experience, thanks to:
- Casey Martin 
    - CPU + MoBo + finding good deals on parts 
    - Answering question after question, and pointing me in the right direction since grad school
- Jonah DeVaney
    - Helping me learn to code on the Node 2 macbook air back in college
    - Pointing me in the right direction since undergrad
- Ryan Redman
    - Giving me the idea for a low performance compute cluster in the first place
    - Making tasty chili
