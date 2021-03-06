.. _qvm:

The Quantum Virtual Machine (QVM)
=================================

The Rigetti Quantum Virtual Machine is an implementation of the Quantum Abstract Machine from
*A Practical Quantum Instruction Set Architecture*. [1]_  It is implemented in ANSI Common LISP and
executes programs specified in the Quantum Instruction Language (Quil). Quil is an opinionated
quantum instruction language: its basic belief is that in the near term quantum computers will
operate as coprocessors, working in concert with traditional CPUs.  This means that Quil is
designed to execute on a Quantum Abstract Machine that has a shared classical/quantum architecture
at its core. The QVM is a wavefunction simulation of unitary evolution with classical control flow
and shared quantum classical memory.

Most API keys give access to the QVM with up to 30 qubits. If you would like access to more qubits
or help running larger jobs, then contact us at support@rigetti.com. On request we may also
provide access to a QVM that allows persistent wavefunction memory between different programs as
well as direct access to the wavefunction memory (wrapped as a ``numpy`` array) from python.

.. _qvm_use:

Using the QVM
-------------
The QVM is available in pyQuil via the ``api`` module.

.. code:: python

    from pyquil.api import QVMConnection
    qvm = QVMConnection()

One executes quantum programs on the QVM using two paradigms: the ``.run(...)`` method, and
the ``.wavefunction(...)`` method. The former closely mirrors how one will execute programs on a
real QPU (see :ref:`qpu_use`), while the latter takes advantage of the virtual machine, and allows
direct access to the wavefunction. These two methods are described in the following two sections.
(For information on constructing quantum programs, please refer back to :ref:`basics`.)

The ``.run(...)`` method
~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: python

    program = Program(X(0), MEASURE(0, 0))
    results = qvm.run(program, trials=1)
    # results = [[1]]

The ``.run(...)`` method takes numerous arguments, several of which are optional. The most important
are

1. the ``program`` to be executed on the QVM,
2. the ``classical_addresses`` which to be returned from the QVM (not included above; by default, these are set to the addresses used in the program's ``MEASURE`` instructions), and
3. the number of ``trials`` to be executed on the machine.

The results returned are a *list of lists of integers*. In the above case, that's

.. parsed-literal::

    [[1]]

Let's unpack this. The *outer* list is an
enumeration over the trials; if you set ``trials=1`` then ``len(results)`` should equal ``1``.

The *inner* list, on the other hand, is an enumeration over the results stored in the classical
addresses. We see that the result of this program is that the classical register ``[0]`` now stores
the state of qubit 0, which should be ``1`` after an :math:`X`-gate. We
can of course ask for more classical registers:

.. code:: python

    qvm.run(p, [0, 1, 2])

.. parsed-literal::

    [[1, 0, 0]]

The classical registers are initialized to zero, so registers ``[1]``
and ``[2]`` come out as zero. If we stored the measurement in a
different classical register we would obtain:

.. code:: python

    p = Program()   # clear the old program
    p.inst(X(0)).measure(0, 1)
    qvm.run(p, [0, 1, 2])

.. parsed-literal::

    [[0, 1, 0]]

We can also run programs multiple times and accumulate all the results
in a single list.

.. code:: python

    coin_flip = Program().inst(H(0)).measure(0, 0)
    num_flips = 5
    qvm.run(coin_flip, [0], num_flips)

.. parsed-literal::

    [[0], [1], [0], [1], [0]]

Try running the above code several times. You will see that you will,
with very high probability, get different results each time.

The ``.wavefunction(...)`` method
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The QVM is a virtual machine. As such, we can directly inspect the wavefunction of
a program, even without measurements, using the ``.wavefunction(...)`` method:

.. code:: python

    coin_flip = Program().inst(H(0))
    qvm.wavefunction(coin_flip)

.. parsed-literal::

    <pyquil.wavefunction.Wavefunction at 0x1088a2c10>

The return value is a Wavefunction object that stores the amplitudes of the
quantum state at the conclusion of the program. We can print this object

.. code:: python

    coin_flip = Program().inst(H(0))
    wavefunction = qvm.wavefunction(coin_flip)
    print(wavefunction)

.. parsed-literal::

  (0.7071067812+0j)|0> + (0.7071067812+0j)|1>

To see the amplitudes listed as a sum of computational basis states. We can index into those
amplitudes directly or look at a dictionary of associated outcome probabilities.

.. code:: python

  assert wavefunction[0] == 1 / np.sqrt(2)
  # The amplitudes are stored as a numpy array on the Wavefunction object
  print(wavefunction.amplitudes)
  prob_dict = wavefunction.get_outcome_probs() # extracts the probabilities of outcomes as a dict
  print(prob_dict)
  prob_dict.keys() # these stores the bitstring outcomes
  assert len(wavefunction) == 1 # gives the number of qubits

.. parsed-literal::

  [ 0.70710678+0.j  0.70710678+0.j]
  {'1': 0.49999999999999989, '0': 0.49999999999999989}

The result from a wavefunction call also contains an optional amount of classical memory to
check:

.. code:: python

    coin_flip = Program().inst(H(0)).measure(0,0)
    wavefunction = qvm.wavefunction(coin_flip, classical_addresses=range(9))
    classical_mem = wavefunction.classical_memory

Additionally, we can pass a random seed to the Connection object. This allows us to reliably
reproduce measurement results for the purpose of testing:

.. code:: python

    seeded_cxn = api.QVMConnection(random_seed=17)
    print(seeded_cxn.run(Program(H(0)).measure(0, 0), [0], 20))

    seeded_cxn = api.QVMConnection(random_seed=17)
    # This will give identical output to the above
    print(seeded_cxn.run(Program(H(0)).measure(0, 0), [0], 20))

It is important to remember that this ``wavefunction`` method is just a useful debugging tool
for small quantum systems, and it cannot be feasibly obtained on a
quantum processor.

.. _basis-ordering:

Multi-Qubit Basis Enumeration
-----------------------------

The Rigetti QVM enumerates bitstrings such that qubit `0` is the least significant bit (LSB)
and therefore on the right end of a bitstring as shown in the table below which contains some
examples.

=============== ============= ===== ========= ========= =========
 bitstring       qubit_(n-1)   ...   qubit_2   qubit_1   qubit_0
=============== ============= ===== ========= ========= =========
  1...101                  1   ...         1         0         1
  0...110                  0   ...         1         1         0
=============== ============= ===== ========= ========= =========



This convention is counter to that often found in the quantum computing literature where
bitstrings are often ordered such that the lowest-index qubit is on the left.
The vector representation of a wavefunction assumes the "canonical" ordering of basis elements.
I.e., for two qubits this order is ``00, 01, 10, 11``.
In the typical Dirac notation for quantum states, the tensor product of two different degrees of
freedom is not always explicitly understood as having a fixed order of those degrees of freedom.
This is in contrast to the kronecker product between matrices which uses the same mathematical
symbol and is clearly not commutative.
This, however, becomes important when writing things down as coefficient vectors or matrices:

.. math::

    \ket{0}_0 \otimes \ket{1}_1 = \ket{1}_1 \otimes \ket{0}_0
    = \ket{10}_{1,0} \equiv \begin{pmatrix} 0 \\ 0 \\ 1 \\ 0 \end{pmatrix}

As a consequence there arise some subtle but important differences in the ordering of wavefunction
and multi-qubit gate matrix coefficients.
According to our conventions the matrix

.. math::

    U_{\rm CNOT(1,0)} \equiv
    \begin{pmatrix}
        1 & 0 & 0 & 0 \\
        0 & 1 & 0 & 0 \\
        0 & 0 & 0 & 1 \\
        0 & 0 & 1 & 0
    \end{pmatrix}

corresponds to the Quil instruction ``CNOT(1, 0)`` which is counter to how most other people in the
field order their tensor product factors (or more specifically their kronecker products).
In this convention ``CNOT(0, 1)`` is given by

.. math::

    U_{\rm CNOT(0,1)} \equiv
    \begin{pmatrix}
        1 & 0 & 0 & 0 \\
        0 & 0 & 0 & 1 \\
        0 & 0 & 1 & 0 \\
        0 & 1 & 0 & 0
    \end{pmatrix}

For additional information why we decided on this basis ordering check out our note
*Someone shouts, "|01000>!" Who is Excited?* [2]_.

.. [1] https://arxiv.org/abs/1608.03355
.. [2] https://arxiv.org/abs/1711.02086

Examples of Quantum Programs
----------------------------

To create intuition for a new class of algorithms, that will run on Quantum Virtual Machines (QVM), it is useful (and
fun) to play with the abstraction that the software provides.

A broad class of programs that can easily be implemented on a QVM are generalizations of
`Game Theory to incorporate Quantum Strategies <https://arxiv.org/abs/quant-ph/0611234>`_.

Meyer-Penny Game
~~~~~~~~~~~~~~~~

A conceptually simple example that falls into this class is the
`Meyer-Penny Game <https://link.aps.org/doi/10.1103/PhysRevLett.82.1052>`_. The game goes as follows: The Starship
Enterprise, during one of its deep-space missions, is facing an immediate calamity, when a powerful alien suddenly
appears on the bridge. The alien, named Q, offers to help Picard, the captain of the Enterprise, under the condition
that Picard beats Q in a simple game of penny flips.

The rules: Picard is to place a penny Heads up into an opaque box. Then Picard and Q take turns to flip or not flip the
penny without being able to see it; first Q then P then Q again. After this the penny is revealed; Q wins if it shows
Heads (H), while Tails (T) makes Picard the winner.

Picard quickly estimates that his chance of winning is 50% and agrees to play the game. He loses the first round and
insists on playing again. To his surprise Q agrees, and they continue playing several rounds more, each of which Picard
loses. How is that possible?

What Picard did not anticipate is that Q has access to quantum tools. Instead of flipping the penny, Q puts the penny
into a superposition of Heads and Tails proportional to the quantum state :math:`|H\rangle+|T\rangle`. Then no matter
whether Picard flips the penny or not, it will stay in a superposition (though the relative sign might change). In the
third step Q undoes the superposition and always finds the penny to shows Heads.

To simulate the game we first construct the corresponding quantum circuit, which takes two qubits -- one to simulate
Picard's choice whether or not to flip the penny and the other to represent the penny. The initial state for all Qubits
is :math:`|0\rangle (= |T\rangle)`. To simulate Picard's decision, we assume that he chooses randomly whether or not to
flip the coin, in agreement with the optimal strategy for the classic penny-flip game. This random choice can be created
by putting one qubit into an equal superposition, e.g. with the Hadamard gate H, and then measure its state. The
measurement will show Heads or Tails with equal probability p=0.5.

To simulate the penny flip game we take the second qubit and put it into its excited state
:math:`|1\rangle (= |H\rangle)` by applying the X (or NOT) gate. Q's first move is to apply the Hadamard gate H.
Picard's decision about the flip is simulated as a CNOT operation where the control bit is the outcome of the random
number generator described above. Finally Q applies a Hadamard gate again, before we measure the outcome. The full
circuit is shown in the figure below.

.. figure:: images/MeyerPennyGame.png
    :align: center
    :figwidth: 65%

First we import all the necessary tools:

.. code-block:: python

    from pyquil.quil import Program
    import pyquil.api as api

    from pyquil.gates import I, H, X
    qvm = api.QVMConnection()

Then we need to define two registers that will be used for the measurement of Picard's decision bit and the final answer
of the penny tossing game.

.. code-block:: python

    picard_register = 1
    answer_register = 0

Moreover we need to encode the two different actions of Picard, which conceptually is equivalent to an `if-else` control
flow as:

.. code-block:: python

    then_branch = Program(X(0))
    else_branch = Program(I(0))


and then wire it all up into the overall measurement circuit:

.. code-block:: python

   prog = (Program()
       # Prepare Qubits in Heads state or superposition, respectively
       .inst(X(0), H(1))
       # Q puts the penny into a superposition
       .inst(H(0))
       # Picard makes a decision and acts accordingly
       .measure(1, picard_register)
       .if_then(picard_register, then_branch, else_branch)
       # Q undoes his superposition operation
       .inst(H(0))
       # The outcome is recorded into the answer register
       .measure(0, answer_register))


Finally we play the game several times

.. code-block:: python

   qvm.run(prog, [0, 1], trials=10)


and record the register outputs as

.. code-block:: python

   [[1, 1],
    [1, 1],
    [1, 0],
    [1, 0],
    [1, 0],
    [1, 0],
    [1, 1],
    [1, 1],
    [1, 0],
    [1, 0]]

Remember that the first number is the outcome of the game (value of the `answer_register`) whereas the second number is
the outcome of Picard’s decision (value of the `picard_register`).

Indeed, no matter what Picard does, Q will always win!

Exercises
~~~~~~~~~

Prisoner's Dilemma
__________________

A classic strategy game is the `prisoner's dilemma <https://en.wikipedia.org/wiki/Prisoner%27s_dilemma>`_ where two
prisoners get the minimal penalty if they collaborate and stay silent, get zero penalty if one of them defects and the
other collaborates (incurring maximum penalty) and get intermediate penalty if they both defect. This game has an
equilibrium where both defect and incur intermediate penalty.

However, things change dramatically when we allow for quantum strategies leading to the
`Quantum Prisoner's Dilemma <https://arxiv.org/abs/quant-ph/9806088>`_.

Can you design a program that simulates this game?
