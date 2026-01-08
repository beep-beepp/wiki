---
title: "Method Area"
parent: JVM
grand_parent: Java
permalink: /java/jvm/method-area
---

# JVM 내부 구조

![img.png](img/img.png)
[Class Loader](class-loader.md)

# Runtime Data Area
JVM은 프로그램 실행되는 동안 사용되는 다양한 run-time data areas를 정의한다.
<br>이 중 일부 data area는 JVM 시작시 생성되고 JVM이 종료될 때만 종료된다.
<br>다른 data area는 스레드마다 있다. 스레드마다 있는 data area는 스레드가 생성될때 생성되고 스레드가 종료될 때 종료된다.


# Components
- PC Register
- JVM Stacks
- Heap
- Method Area
- Run-Time Constant Pool
- Native Method Stack


# PC Register
JVM은 동시에 많은 실행 스레드를 지원할 수 있다.
<br>각 JVM 스레드는 자신만의 pc(program counter) register를 가진다.
<br>어느 시점에서든, 각 JVM 스레드는 단 하나의 메서드의 코드를 실행하고 있으며, 그 메서드는 해당 스레드의 현재 메서드이다.
<br>만약 그 메서드가 native method가 아니라면, pc register에는 현재 실행 중인 JVM 명령어의 주소가 들어있다.
<br>반대로 해당 native method라면, pc register값은 undefined다.
<br>JVM의 pc register는 해당 플랫폼(JVM이 실행중인 플랫폼) 에서 returnAddress나 native pointer를 담을 수 있을 만큼 충분히 크다.


# JVM Stacks
각 JVM 스레드는 스레드가 생성되는 시점에 함께 생성되는, 자신만의 JVM stack을 가진다.

- JVM Stack은 frame을 저장한다.
- C와 같은 일반적인 언어의 stack과 유사하다. 즉, 로컬 변수와 중간 계산 결과(partial results)를 보관하며, 메서드 호출과 반환 과정에서 역할을 수행한다.
- JVM Stack은 frame을 push하거나 pop하는 경우를 제외하고는 직접 조작되지 않기 때문에, frame은 heap에 할당될 수 있다.
- JVM Stack을 위한 메모리는 물리적으로 연속된(contiguous) 공간일 필요는 없다.
- JVM 명세에 따르면, JVM Stack은 고정된 크기로 만들 수도, 프로그램 실행 중 필요에 따라 동적으로 확장 및 축소될 수도 있다.
  - 만약 고정된 크기인 경우, 각 Stack의 크기는 스레드가 만들어질 때 개별적으로 정해질 수 있다.
  - JVM 구현체는 프로그래머 또는 사용자에게 JVM Stack의 초기 크기를 설정할 수 있는 기능을, 또한 JVM Stack이 동적으로 확장/축소되는 경우에는 최대 크기와 최소 크기를 설정할 수 있는 기능을 제공할 수도 있다.
- JVM Stack과 관련해 발생할 수 있는 예외 상황
  - 어떤 스레드가 실행되는 동안 허용된 Stack의 크기보다 더 큰 크기를 요구하는 경우, JVM은 StackOverflowError를 던진다.
  - JVM Stack이 동적으로 확장될 수 있는 경우, Stack을 늘리려고 했지만 필요한 메모리를 확보할 수 없거나, 새로운 스레드를 시작하면서 초기 Stack을 생성하는 데 필요한 메모리를 확보할 수 없는 경우, JVM은 OutOfMemoryError를 던진다.


# Heap
JVM은 모든 JVM 스레드가 공유하는 heap을 가진다.
heap은 모든 클래스 인스턴스와 배열이 생성될때 필요한 메모리가 할당되는 run-time data area이다.

- heap은 JVM이 시작될 때 함께 생성된다.
- 객체가 차지하던 메모리는 garbage collector라고 불리는 자동 저장 관리 시스템에 의해 회수되며, 개발자가 객체를 직접 해제하는 일은 없다.
- JVM은 특정한 방식의 자동 저장 관리 시스템을 강요하지 않으며, 어떤 방식을 사용할지는 JVM 구현체가 자신의 시스템 요구 사항에 맞게 선택할 수 있다.
- heap은 고정된 크기일 수도, 프로그램 실행 중 필요해지면 확장될 수도 있고, 더 이상 큰 힙이 필요하지 않게 되면 축소될 수도 있다.
- heap을 위한 메모리는 물리적으로 연속된(contiguous) 공간일 필요는 없다.
- JVM 구현체는 프로그래머 또는 사용자에게 heap의 초기 크기를 설정할 수 있는 기능을, heap이 동적으로 확장/축소될 경우에는 최대 힙 크기와 최소 힙 크기에 역시 설정할 수 있는 기능을 제공할 수도 있다.
- heap과 관련해 발생할 수 있는 예외 상황
  - 어떤 계산이 자동 저장 관리 시스템이 제공할 수 있는 것보다 더 많은 힙 메모리를 요구하는 경우 JVM은 OutOfMemoryError를 던진다.


# Method Area
JVM은 모든 JVM 스레드가 공유하는 method area를 가진다.

- method area는 일반적인 프로그래밍 언어에서 컴파일된 코드 저장되는 공간이나, 운영체제 프로세스에서의 text segment와 비슷한 역할을 한다.
- 이 area엔 각 클래스마다 필요한 정보들이 저장된다. 
  - run-time constant pool
  - 필드와 메서드에 대한 메타데이터
  - 메서드와 생성자의 실제 실행 코드 
  - 또한 클래스 및 인터페이스가 처음 사용될 때 수행되는 초기화 코드와, 객체가 생성될 때 실행되는 인스턴스 초기화 코드 역시 포함된다.
- method area은 JVM이 시작될 때 함께 생성된다.
- method area은 논리적으로는 heap의 일부로 취급되지만, 구현이 단순한 JVM의 경우 method area는 garbage collection이나 압축 대상이 아닐 수도 있다.
- JVM 명세에선 method area이 실제로 어디에 위치해야 하는지나 컴파일된 코드를 어떤 방식으로 관리해야 하는지를 강제하지 않는다.
- method area는 고정된 크기일 수도, 프로그램 실행 중 필요에 따라 확장될 수도, 더 이상 큰 method area가 필요하지 않게 되면 축소될 수도 있다.
- method area를 위한 메모리는 물리적으로 연속된(contiguous) 공간일 필요는 없다.
- JVM 구현체는 프로그래머 또는 사용자에게 method area의 초기 크기를 설정할 수 있는 기능을, 또한 method area의 크기가 가변적인 경우에는 최대 method area 크기와 최소 method area 크기를 설정할 수 있는 기능을 제공할 수도 있다.
- method area와 관련해 발생할 수 있는 예외 상황
  - method area에서 어떤 할당 요청을 처리할 만큼의 메모리를 확보할 수 없는 경우 JVM은 OutOfMemoryError를 던진다.


# Run-Time Constant Pool
run-time constant pool은 클래스 파일에 있는 constant_pool 테이블을 클래스나 인터페이스마다 실행 중에 사용할 수 있도록 만든 구조이다.

- run-time constant pool에는 여러 종류의 상수가 포함된다.
  - 컴파일 타임에 이미 값이 정해진 숫자 상수 뿐만 아니라, 실행 중에 실제 대상이 연결되어야 하는 메서드 참조와 필드 참조까지 포함된다.
- 일반적인 프로그래밍 언어에서의 symbol table과 비슷한 역할을 하지만, 단순한 이름 정보만 담는 일반적인 symbol table보다 더 넓은 범위의 데이터를 포함한다.
- 각 run-time constant pool은 JVM의 method area로부터 할당받아 생성된다.
- 클래스 또는 인터페이스에 대한 run-time constant pool은 JVM에 의해 해당 클래스 또는 인터페이스가 생성될 때 구성된다.
- 클래스 또는 인터페이스에 대한 run-time constant pool의 구성과 관련해 발생할 수 있는 예외 상황
  - 클래스 또는 인터페이스를 생성하는 과정에서, run-time constant pool을 만들기 위해 method area에서 제공할 수 있는 메모리보다 더 많은 메모리가 필요한 경우 JVM은 OutOfMemoryError를 던진다. 


# Native Method Stacks
JVM 구현체는 Java가 아닌 다른 언어로 작성된 메서드(native method)를 실행하기 위해, 
일반적으로 "C stacks"라고 불리는 일반적인 stack 구조를 사용할 수 있다. 
이러한 native method stack은 또한 C와 같은 언어로 작성된 JVM interpreter를 구현할 때도 활용될 수 있다.

- 만약 어떤 JVM 구현체가 native method를 로드하지 않고 또한 내부적으로도 일반적인 stack 구조에 의존하지 않는다면, 그 JVM은 native method stack을 제공하지 않아도 된다.
  - native method stack이 제공되는 경우에는, 보통 각 스레드가 생성될 때 스레드마다 하나씩 할당된다.
- 이 명세에선 native method stack이 고정된 크기일 수도 있고, 프로그램 실행 중 필요에 의해 동적으로 확장/축소될 수 있다.
  - 만약 native method stack이 고정된 크기인 경우, 각 native method stack의 크기는 해당 stack이 생성될 때 각각 개별적으로 결정될 수 있다.
- JVM 구현체는 프로그래머 또는 사용자에게 native method stack의 초기 크기를 설정할 수 있는 기능을, 또한 native method stack의 크기가 가변적인 경우에는 최대 native method 스택 크기와 최소 native method stack 크기를 설정할 수 있는 기능을 제공할 수도 있다.
- native method stack과 관련해 발생할 수 있는 예외 상황
  - 어떤 스레드에서 수행되는 계산이 허용된 크기보다 더 큰 native method stack을 요구하는 경우 JVM은 StackOverflowError를 던진다.
  - native method stack이 동적으로 확장 가능한 경우, native method stack을 늘리려 했지만 필요한 메모리를 확보할 수 없는 경우, 또는 새로운 스레드를 생성하면서 초기 native method stack을 만드는데 필요한 메모리를 확보할 수 없는 경우 JVM은 OutOfMemoryError를 던진다.


# 참고
[Java SE 25](https://docs.oracle.com/javase/specs/jvms/se25/jvms25.pdf)
