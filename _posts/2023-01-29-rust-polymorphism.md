---
layout: post
title: Polymorphism in Rust
tags: rust polymorphism trait generics enum
---

안녕하세요. 오늘은 rust에서 polymorphism을 활용하는 방법을 정리해 보고자 합니다.

cpp 코드를 작성하던 습관 대로 Rust 코드를 작성하다 보면 막히는 부분 중 하나가 interface 클래스(pure virtual class)와 이를 통한 객체 전달 부분입니다. 단순히 trait으로 변환하여 코드를 작성하다 보면 쉽게 컴파일 에러 지옥에 빠지곤 하는데요..

아주 간단한 composite design pattern 예제를 구현해 보면서 rust의 polymorphism 에 대해 알아보도록 하겠습니다. Composite pattern 예제로 많이 사용되는 File, Directory 구조를 표현해 보고자 합니다. File은 이름과 크기를 갖고 있는 객체로, Directory는 이름과 하위 파일 및 디렉토리를 갖는 객체로, 그리고 File, Directory 는 모두 Node 라는 interface를 구현하게 만들고자 합니다. 

<center><img src="/assets/images/image05.png"></center>

<br>

File, Directory가 공통으로 구현하고 있는 Interface의 각 함수는 다음과 같은 의미를 갖고 있습니다.
- get_name: 파일 혹은 디렉토리의 이름 반환.
- get_size: 파일의 크기를 반환. 디렉토리의 경우, 갖고 있는 모든 파일 혹은 디렉토리 크기의 합을 반환해야 함.

<br>
<br>

## Trait을 사용한 Composite Pattern 구현
아주 나이브하게는 interface를 trait으로 1:1 변환하여 다음과 같이 구현할 수 있을 것 같습니다.

````rust
fn main() {
    let mut root_node = Directory::new("root".to_string());

    let new_file = File::new("new_file".to_string(), 1024 * 1024);
    let new_directory = Directory::new("my_folder".to_string());

    root_node.add_new_node(new_file);
    root_node.add_new_node(new_directory);

    assert_eq!(root_node.get_size(), 1024 * 1024);
}

trait Node {
    fn get_name(&self) -> String;
    fn get_size(&self) -> u64;
}

struct File {
    name: String,
    size: u64,
}
impl Node for File {
    fn get_name(&self) -> String {
        self.name.clone()
    }

    fn get_size(&self) -> u64 {
        self.size
    }
}
impl File {
    pub fn new(name: String, size: u64) -> Self {
        Self {
            name,
            size,
        }
    }
}

struct Directory {
    name: String,
    childs: Vec<dyn Node>,
}
impl Node for Directory {
    fn get_name(&self) -> String {
        self.name.clone()
    }

    fn get_size(&self) -> u64 {
        self.childs.iter().fold(0, |acc, x| acc + x.get_size())
    }
}
impl Directory {
    pub fn new(name: String) -> Self {
        Self {
            name,
            childs: Vec::new(),
        }
    }
    pub fn add_new_node(&mut self, node: dyn Node) {
        self.childs.push(node);
    }
}
````

이 코드는 사실 빌드 되지는 않습니다. 이유는 다음과 같습니다.
````cpp
error[E0277]: the size for values of type `(dyn Node + 'static)` cannot be known at compilation time
   --> src/bin/composite.rs:42:13
    |
42  |     childs: Vec<dyn Node>,
    |             ^^^^^^^^^^^^^ doesn't have a size known at compile-time
    |
    = help: the trait `Sized` is not implemented for `(dyn Node + 'static)`
note: required by a bound in `Vec`
   --> /Users/huijeongkim/.rustup/toolchains/stable-aarch64-apple-darwin/lib/rustlib/src/rust/library/alloc/src/vec/mod.rs:400:16
    |
400 | pub struct Vec<T, #[unstable(feature = "allocator_api", issue = "32838")] A: Allocator = Global> {
    |                ^ required by this bound in `Vec`

For more information about this error, try `rustc --explain E0277`.
````

`dyn Node`는 compile time에 크기를 알 수 없기 때문에 Vector에 넣을 수 없다는 것 입니다. 이를 아주 간단하게 해결할 수 있는 방법은, 객체 자체가 아닌 객체를 가리키는 포인터를 Vector에 넣는 것 입니다. `dyn Node`가 아닌 Heap으로 옮겨 진 `Box<dyn Node>`의 리스트를 갖도록 수정하면 에러는 사라지고 컴파일 성공하게 됩니다.

````rust
// main 함수 수정
root_node.add_new_node(Box::new(new_file));
root_node.add_new_node(Box::new(new_directory));

// Directory 수정
struct Directory {
    name: String,
    childs: Vec<Box<dyn Node>>,
}

impl Directory {
    pub fn add_new_node(&mut self, node: Box<dyn Node>) {
        self.childs.push(node);
    }
}
````

<br>
<br>

## Trait Clone

여기서 File, Directory 를 clone 하도록 수정해 보겠습니다. `#[derive(Clone)]` 매크로를 사용하면 손쉽게 Clone trait을 구현할 수 있습니다. 하지만 여기서 다시 한 번 컴파일 에러를 만나게 됩니다. 

 ````cpp
error[E0277]: the trait bound `dyn Node: Clone` is not satisfied
  --> src/bin/composite.rs:51:5
   |
48 | #[derive(Clone)]
   |          ----- in this derive macro expansion
...
51 |     childs: Vec<Box<dyn Node>>,
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `Clone` is not implemented for `dyn Node`
   |
   = note: required because of the requirements on the impl of `Clone` for `Box<dyn Node>`
   = note: 1 redundant requirement hidden
   = note: required because of the requirements on the impl of `Clone` for `Vec<Box<dyn Node>>`
   = note: this error originates in the derive macro `Clone` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0277`.
 ````

`dyn Node`에 대한 clone 이 구현되어 있지 않다는 것 입니다. Directory가 clone 가능하려면 그 내부의 모든 멤버도 clone 가능해야 합니다. 따라서 `dyn Node`의 clone을 구현해 줘야 합니다.

derive 매크로는 struct, enum, union에만 사용 가능하므로 `trait Node`에는 `#[derive(Clone)]`을 추가할 수 없습니다. 대신 다음과 같이 구현할 수 있습니다. 참고로 clone 함수의 리턴이 `dyn Node`가 되면 첫 번째 에러와 동일한 이유, compile time에 size를 알 수 없다는 문제로 에러가 발생하기 때문에 이 또한 Box 로 clone하도록 구현하였고, 이 때문에  `Box<dyn Node>`에 대한 clone 구현도 추가하였습니다.

````rust
// Box<dyn Node> 에 대한 clone을 추가
trait Node : CloneNode {
    fn get_name(&self) -> String;
    fn get_size(&self) -> u64;
}
trait CloneNode {
    fn clone_box(&self) -> Box<dyn Node>;
}
impl Clone for Box<dyn Node> {
    fn clone(&self) -> Self {
        self.clone_box()
    }
}

// File에 CloneNode 추가
#[derive(Clone)]
struct File {
    name: String,
    size: u64,
}
impl CloneNode for File {
    fn clone_box(&self) -> Box<dyn Node> {
        Box::new(self.clone())
    }
}

// Directory에 CloneNode 추가
#[derive(Clone)]
struct Directory {
    name: String,
    childs: Vec<Box<dyn Node>>,
}
impl CloneNode for Directory {
    fn clone_box(&self) -> Box<dyn Node> {
        Box::new(self.clone())
    }
}
````

`clone_box` 함수를 만드는 패턴은 생각보다 자주 반복 사용되어 boilerplate 코드가 되기도 합니다. 이럴 때 [dyn_clone](https://docs.rs/dyn-clone/latest/dyn_clone/) crate를 사용하면 이 코드들을 제거할 수 있습니다.

````rust
use dyn_clone::DynClone;
trait Node : DynClone {
    fn get_name(&self) -> String;
    fn get_size(&self) -> u64;
}
dyn_clone::clone_trait_object!(Node);
````

이렇게 수정하면 위에서 구현했던 `CloneNode` trait이나 `clone_box` 구현을 생략할 수 있습니다.

<br>
<br>

## Generic을 활용하기
Rust 에서 polymorphism을 활용하는 또 다른 방법은 Generic을 사용하는 것 입니다. Directory에 적용해 보면 다음과 같습니다.

````rust
#[derive(Clone)]
struct Directory<N: Node> {
    name: String,
    childs: Vec<Box<N>>,
}
impl<N: Node + Clone> Node for Directory<N> {
    fn get_name(&self) -> String {
        self.name.clone()
    }

    fn get_size(&self) -> u64 {
        self.childs.iter().fold(0, |acc, x| acc + x.get_size())
    }
}
impl<N: Node> Directory<N> {
    pub fn new(name: String) -> Self {
        Self {
            name,
            childs: Vec::new(),
        }
    }
    pub fn add_new_node(&mut self, node: Box<N>) {
        self.childs.push(node);
    }
}
````

이렇게 작성했을 때의 문제는 `N: Node`가 File 혹은 Directory 중 하나만 될 수 있다는 것 입니다. main 함수에서 다음과 같이 `add_new_node`를 File, Directory 에 대해 호출한다면,

````rust
root_node.add_new_node(Box::new(new_file));
root_node.add_new_node(Box::new(new_directory));
````

다음과 같이 컴파일 에러가 발생하게 됩니다.
````cpp
error[E0308]: mismatched types
   --> src/bin/composite.rs:10:37
    |
10  |     root_node.add_new_node(Box::new(new_directory));
    |                            -------- ^^^^^^^^^^^^^ expected struct `File`, found struct `Directory`
    |                            |
    |                            arguments to this function are incorrect
    |
    = note: expected struct `File`
               found struct `Directory<_>`
note: associated function defined here
   --> /Users/huijeongkim/.rustup/toolchains/stable-aarch64-apple-darwin/lib/rustlib/src/rust/library/alloc/src/boxed.rs:213:12
    |
213 |     pub fn new(x: T) -> Self {
    |            ^^^

For more information about this error, try `rustc --explain E0308`.
````

구체 객체가 run-time에 결정되어야 하는 이런 예제에서는 generic은 적합하지 않은 것이죠. 이 문제를 해결하기 위해서 enum을 사용해 볼 수 있겠습니다.

<br>
<br>

## Enum으로 감싸기
Rust 의 enum 또한 polymorphism을 활용할 수 있는 방법 중 하나입니다. 다음과 같이 trait의 구현체들을 enum value로 넣어줄 수 있습니다.

````rust
// enum을 하나 추가하고
#[derive(Clone)]
enum NodeType {
    FileNode(File),
    DirectoryNode(Directory),
}

// Directory는 boxed trait 대신 enum을 갖도록 수정,
#[derive(Clone)]
struct Directory {
    name: String,
    childs: Vec<NodeType>,
}

// main 함수에서는 다음과 같이 node를 추가합니다.
root_node.add_new_node(NodeType::FileNode(new_file));
root_node.add_new_node(NodeType::DirectoryNode(new_directory));
````

이 떄 enum 내의 value를 꺼내서 trait 함수를 부르기 번거로우므로, enum에도 `Node` trait을 구현해 줍니다.
````rust
impl Node for NodeType {
    fn get_name(&self) -> String {
        match &self {
            NodeType::FileNode(f) => f.get_name(),
            NodeType::DirectoryNode(d) => d.get_name(),
        }
    }

    fn get_size(&self) -> u64 {
        match &self {
            NodeType::FileNode(f) => f.get_size(),
            NodeType::DirectoryNode(d) => d.get_size(),
        }
    }
}
````

이런 패턴 또한 굉장히 많이 사용되는 것 같습니다. 이 코드도 [enum_dispatch](https://docs.rs/enum_dispatch/0.3.11/enum_dispatch/) crate를 사용하면 단순화할 수 있습니다.

````rust
use enum_dispatch::enum_dispatch;

#[enum_dispatch]
trait Node : DynClone {
    fn get_name(&self) -> String;
    fn get_size(&self) -> u64;
}
dyn_clone::clone_trait_object!(Node);

#[derive(Clone)]
#[enum_dispatch(Node)]
enum NodeType {
    FileNode(File),
    DirectoryNode(Directory),
}
````
<br>
<br>

## 최종 코드

여태까지 수정한 내용을 모두 반영한 최종 코드는 다음과 같습니다.

````rust
fn main() {
    let mut root_node = Directory::new("root".to_string());

    let new_file = File::new("new_file".to_string(), 1024 * 1024);
    let new_directory = Directory::new("my_folder".to_string());

    root_node.add_new_node(NodeType::FileNode(new_file));
    root_node.add_new_node(NodeType::DirectoryNode(new_directory));

    assert_eq!(root_node.get_size(), 1024 * 1024);
}

use dyn_clone::DynClone;
use enum_dispatch::enum_dispatch;

#[enum_dispatch]
trait Node : DynClone {
    fn get_name(&self) -> String;
    fn get_size(&self) -> u64;
}
dyn_clone::clone_trait_object!(Node);

#[derive(Clone)]
#[enum_dispatch(Node)]
enum NodeType {
    FileNode(File),
    DirectoryNode(Directory),
}

#[derive(Clone)]
struct File {
    name: String,
    size: u64,
}
impl Node for File {
    fn get_name(&self) -> String {
        self.name.clone()
    }

    fn get_size(&self) -> u64 {
        self.size
    }
}
impl File {
    pub fn new(name: String, size: u64) -> Self {
        Self {
            name,
            size,
        }
    }
}

#[derive(Clone)]
struct Directory {
    name: String,
    childs: Vec<NodeType>,
}
impl Node for Directory {
    fn get_name(&self) -> String {
        self.name.clone()
    }

    fn get_size(&self) -> u64 {
        self.childs.iter().fold(0, |acc, x| acc + x.get_size())
    }
}
impl Directory {
    pub fn new(name: String) -> Self {
        Self {
            name,
            childs: Vec::new(),
        }
    }
    pub fn add_new_node(&mut self, node: NodeType) {
        self.childs.push(node);
    }
}
````

<br>
<br>

이렇게 아주 간단한 composite pattern 예제를 rust로 구현해 봤습니다. 매 번 헷갈리던 거여서 정리해 봤는데 이젠 그만 헷갈리길...

<br>
<br>