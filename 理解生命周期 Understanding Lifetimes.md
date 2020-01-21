# 理解生命周期 Understanding Lifetimes（转）

原文地址：
[https://rniczh.github.io/blog/lifetimes-intro/](https://rniczh.github.io/blog/lifetimes-intro/)
原文发布时间为2019-06-05

The Lifetimes in this post is based on the 2018 edition, neither 2015 edition nor the new Borrow checker will be applied in the furture[1](https://rniczh.github.io/blog/lifetimes-intro/#org979af84).
In fact, you can just go to read the [RFC](https://github.com/rust-lang/rfcs/blob/master/text/2094-nll.md) to understand how the borrowck work currently. And in this post, I want to describe the Lifetimes in a different way that what I’m learned from the RFC.
**Audience**: You may already have read the **Rust Book**. Nice if you took a compiler course.
> By the way, I have speaked on this topic in two meetups; Cat system workshop and Rust taiwan meetup, you can find these two presentation at [slide1](https://rniczh.github.io/slides/How-NLL-make-life-easier/#1) and [slide2](https://github.com/rniczh/slides/blob/gh-pages/Lifetimes-intro/lifetimes-intro.pdf), respectively.

## Overview of Borrow checker
The whole borrowck can divided into these four steps.

1. Building Liveness[2](https://rniczh.github.io/blog/lifetimes-intro/#org23a4454)
1. Solving Constraints
1. Compute Loans in Scope
1. Check actions and report errors

In the first two steps, it will compute the Lifetimes. Then bring this information to construct the Loan and compute which Loans are live at which points in CFG in the third step. In the forth, it will traverse through the CFG to check whether each action is legal or not, just depend on the Loans we computed before, but this step is not mentioned in this post, you can read it in [Reporting Erros in RFC](https://github.com/rust-lang/rfcs/blob/master/text/2094-nll.md#borrow-checker-phase-2-reporting-errors), however, it’s quite intuitive for us.
Before continuing, I need to bring some background knowledge in advance.
### Lifetimes (often called Regions)
A Lifetimes is a set of program points on CFG(Control Flow Graph).
For instance, the `let r:&i32 = &x;` can be expanded to `let r: &'0 i32 = &'1 x;` by the compiler, then we can get some fact from this borrow expression:

- Generate a Region `'1` (region variable).
- Create a Constraints: `('1 : '0) @ P`. A Subtyping relation[3](https://rniczh.github.io/blog/lifetimes-intro/#org7d7ede4) with location(program point) information; It’s mean the region `'1` must include all points in `'0` which are rechable from location 𝑃P.

The reason why it’s just generate a region is because you should decompose this borrow expression into the two statements. One is declarative of `r`, and another is an assignment.
```rust
let r: &'0 i32;
r = &'1 x;
```
### Borrow
Each Borrow expression is corresponding to a **Loan** which is a struct for recording some information about this borrow.
Such like `p = &foo;` may create a Loan as shown below.
```rust
Loan L0 {
   point: A/0
   path: { foo },
   kind: shared
   region: '1 { A/1, B/0, C/0 }
}
```
That’s mean a borrow expression happen at `A/0` (the 0th statement of Basic Block A), `foo` is borrowed, create a shared reference tagged with a Lifetimes `'1` (a set of program points computed in the first two steps in borrowck).
### Data Flow Analysis
The process of building liveness in the first step and determine which Loans are live at which points in the third step, are actually the data flow problems[4](https://rniczh.github.io/blog/lifetimes-intro/#org7dd6785).
![](https://cdn.nlark.com/yuque/0/2019/png/476994/1569657587861-0ca269af-b05c-4a0a-bb46-79767a575f7a.png#align=left&display=inline&height=392&originHeight=392&originWidth=451&size=0&status=done&width=451)
Figure 1: data-flow
The Data flow problem come into two flavors: forward or backward direction. Take forward as an exmaple, in Figure 1, we can formulate it with
𝑂𝑢𝑡[𝑠]=𝑔𝑒𝑛𝑠∪(𝐼𝑛[𝑠]∩𝑘𝑖𝑙𝑙𝑠⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯)Out[s]=gens∪(In[s]∩kills¯)
#### Definition

- Input of node `s`: 𝐼𝑛[𝑠]In[s].
- Output: 𝑂𝑢𝑡[𝑠]Out[s].
- Transfer function: 𝑔𝑒𝑛gen and 𝑘𝑖𝑙𝑙kill.

If `p` is a predecessors of `s`, then 𝑂𝑢𝑡[𝑝]=𝐼𝑛[𝑠]Out[p]=In[s]
#### Behavior
The result of 𝑂𝑢𝑡[𝑠]Out[s] is { 𝐼𝑛[𝑠]In[s] substract the 𝑘𝑖𝑙𝑙𝑠kills set and then union with the 𝑔𝑒𝑛𝑠gens set }.
And it will be implemented in a fixed-point iteration alogrithm; That’s mean, it will try to recompute it, until the whole states are fixed. In other words, the 𝐼𝑛[𝑠],∀𝑠∈𝑑𝑜𝑚𝑎𝑖𝑛𝑉In[s],∀s∈domainV will not changed anymore.
## Step1. Building Liveness

- Liveness: a variable 𝑣v is live at point 𝑃P if there exists a path in CFG from 𝑃P to a use of 𝑣v along which 𝑣v is not defined.
### Liveness analysis (Backward data flow problem)

---

Data flow equation
𝐿𝑖𝑣𝑒𝑖𝑛[𝑒𝑥𝑖𝑡]=∅Livein[exit]=∅
𝐿𝑖𝑣𝑒𝑖𝑛[𝑠]=𝑢𝑠𝑒𝑠∪(𝐿𝑖𝑣𝑒𝑜𝑢𝑡[𝑠]∩𝑑𝑒𝑓𝑠⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯)Livein[s]=uses∪(Liveout[s]∩defs¯)
𝐿𝑖𝑣𝑒𝑜𝑢𝑡[𝑡]=⋃𝑠∈𝑠𝑢𝑐𝑐(𝑡)𝐿𝑖𝑣𝑒𝑖𝑛[𝑠]Liveout[t]=⋃s∈succ(t)Livein[s]

---

When we meet a statement like: `a = b + c`, then 𝑑𝑒𝑓𝑠defs is `{ a }`, 𝑢𝑠𝑒𝑠uses is `{b, c}`. If the 𝐿𝑖𝑣𝑒𝑜𝑢𝑡[𝑠]Liveout[s]is `{a, d}` then
𝐿𝑖𝑣𝑒𝑖𝑛[𝑠]={{𝑎,𝑑}−{𝑎}}∪{𝑏,𝑐}={𝑑,𝑏,𝑐}Livein[s]={{a,d}−{a}}∪{b,c}={d,b,c}
For the variable `b`, we can say that `b` is live at location `s` (in 𝑙𝑖𝑣𝑒𝑖𝑛[𝑠]livein[s]), but not live anymore after the location `s`. So the liveness of `b` include points `s` (𝑠∈𝑙𝑖𝑣𝑒𝑛𝑒𝑠𝑠(𝑏)s∈liveness(b)).
### Note
But here is a little different is we only compute the liveness of variable if it has a region associated. For instance, we only care about the `p` that assoicated with region `'0` in the code below. But the liveness of variable `foo` and `bar` just doesn’t matter.
```rust
let foo = 10;
let bar = 20;
let p:&'0 T;
p = &'1 foo;
```
And the liveness of `p` will be assigned to region `'0`. That we say: `'0 = liveness(p)`.
## Step2. Solving Contraints
In Step2, we need to sovle the contraints; In other words, complete the requirements of constraints. After we sovled the constraints, we can get another regions.

- Constraints: a form of `('a : 'b) @ P`; Lifetimes `'a` must include all points in `'b` **which are rechable from the location 𝑃P.**

For instance, in Figure 2, you may already have region `'b = {n1, n3, n4}` which is computed in Step1, and after solved the constraint `('a : 'b) @ P`, you will get the region `'a` which is include `{n1, n4}`.
**![](https://cdn.nlark.com/yuque/0/2019/png/476994/1569657586944-48181255-2f5a-490a-ab18-6ff09cdadb20.png#align=left&display=inline&height=595&originHeight=595&originWidth=1362&size=0&status=done&width=1362)**
**Figure 2: solve constraints**
## **Step3. Compute Loans in Scope**
Each Borrow expression is corresponding to a Loan, that we already talked in [Borrow](https://rniczh.github.io/blog/lifetimes-intro/#borrow).
Determine which Loans are live at which points is also a data flow problem, but unlike the liveness is a backward analysis, it’s a forward.

---

**Data flow equation**
**𝐿𝑜𝑎𝑛𝑜𝑢𝑡[𝑒𝑛𝑡𝑟𝑦]=∅Loanout[entry]=∅**
**𝐿𝑜𝑎𝑛𝑜𝑢𝑡[𝑠]=𝑔𝑒𝑛𝑠∪(𝐿𝑜𝑎𝑛𝑖𝑛[𝑠]∩𝑘𝑖𝑙𝑙𝑠⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯)Loanout[s]=gens∪(Loanin[s]∩kills¯)**
**𝐿𝑜𝑎𝑛𝑖𝑛[𝑠]=⋃𝑝∈𝑝𝑟𝑒𝑑(𝑠)𝐿𝑜𝑎𝑛𝑜𝑢𝑡[𝑝]Loanin[s]=⋃p∈pred(s)Loanout[p]**

- **Transfer functions:**
**𝑔𝑒𝑛gen if the statement at location `s` is a borrow expression.**
**𝑘𝑖𝑙𝑙kill if the 𝐿𝑉LV of the assignment is in path (𝐿𝑉∈𝑝𝑎𝑡ℎLV∈path), or the location `s` in not live here (𝑠∉𝑟𝑒𝑔𝑖𝑜𝑛s∉region).**


---

It is like having lots of flyer(Loans) in your hand, you will copy these flyers and transfer to the next people. If you encounter a branch, just give them to all branches. The next people will check if they should discard some flyers because of the reason above, and then check if they should generate a new flyer because they are a borrow expression, and then transfer to the next people again.
Until the whole Loans at each locatoin are fixed, not changed anymore, then the job is completed.
**![](https://cdn.nlark.com/yuque/0/2019/png/476994/1569657587962-d352b791-7d7c-427b-8381-ede1fa08dfaa.png#align=left&display=inline&height=326&originHeight=326&originWidth=628&size=0&status=done&width=628)**
**Figure 3: Transfer the Loans**
## **Examples**
The code below cannot compiled, because even the L1 be killed on the true path of if-branch, but it’s still live on the other path. So when you borrow the list at line-11, it will invalidate the L1. Some other examples I present in this [slide](https://github.com/rniczh/slides/blob/gh-pages/Lifetimes-intro/lifetimes-intro.pdf).

```rust
let mut list: &mut List = &mut a;
// gen L0
let r = &mut (*list).value;
// gen L1
if (*list).len > 0 {
  list = &mut b;
  // kill L1
  // gen L2
}
// L0, L1, L2 are live here
println!("{:?}", list); // Invalidate L1
println!("{:?}", r);
```

You may wondering why the line-11 will invalidate the L1, that because L1 is just the fact of Borrow expression at line-2, what is it mutable borrow are `(*list).value, (*list), list` by Supporting Prefixes[5](https://rniczh.github.io/blog/lifetimes-intro/#orgc801818). So you can not immutable borrow `list` at line-11.
However, the following code can be allowed by the Compiler, and the reason is quite intuitive.

```rust
let mut list: &mut List = &mut a;
// gen L0
let r = &mut (*list).value;
// gen L1
list = &mut b;
// kill L1
// gen L2
// L0, L2 are live here
println!("{:?}", list);
// kill L0 and L2, because these region not include here
println!("{:?}", r);
```


## **Reference**

1. ** The new Borrowck called Polonius; use `RUSTFLAGS=-Zpolonius cargo +nightly run` to play this new design.**
1. ** Liveness analysis; Also called live variable analysis, mostly used before register allocation in compiler, [wiki](https://en.wikipedia.org/wiki/Live%5Fvariable%5Fanalysis).**
1. ** Subtyping relation. [https://doc.rust-lang.org/nomicon/subtyping.html](https://doc.rust-lang.org/nomicon/subtyping.html), the outlive relation between lifetimes, but it will become subset relation in the furture(Polonius).**
1. ** Data Flow Analysis; The details of this technique can be found in chapter 9 of Compilers : principles, techniques, and tool(2nd), [wiki](https://en.wikipedia.org/wiki/Data-flow%5Fanalysis).**
1. ** Supporting Prefixes; [https://github.com/rust-lang/rfcs/blob/master/text/2094-nll.md#reborrow-constraints](https://github.com/rust-lang/rfcs/blob/master/text/2094-nll.md#reborrow-constraints)**
