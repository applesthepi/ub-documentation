.. role:: cpp(code)
   :language: c++

Cappuccino - OUTDATED
=====================

Disclaimer
----------

This documentation is old and outdated and will only stray further the more the build system gets updated.

About
-----

**Cappuccino** is a library that compiles, debugs, and runs UB programs. During early development **Cappuccino** did not exist. Since we started to really crack down on performance, we needed to create a dedicated runtime environment.

Preprocessor
------------

The preprocessor stage takes place on the executable. Going through all blocks, it creates a ``ModBlockData`` for each one. The ``ModBlockData`` contains the following items:

.. code:: c++

	enum class ModBlockDataType
	{
		VAR, RAW
	};

	enum class ModBlockDataInterpretation
	{
		TEXT, STRING, REAL, BOOL
	};

    std::vector<void*> m_data;
    std::vector<void*> m_preData;
    std::vector<ModBlockDataType> m_types;
    std::vector<ModBlockDataInterpretation> m_interpretations;

``std::vector<void*> m_data;``
	Every argument has one index inside this vector. All arguments that are in variable-mode relay are ``std::string*``. Reals use ``double*`` and booleans use ``bool*``. This is the most basic form of information you can get from each block in the editor.

``std::vector<void*> m_preData;``
	Predata is data that can be transferred from the initialization stages to the runtime stages. It is part of ``ModBlockData`` because this is how the ``ModBlockPass`` gets its information. Predata does nothing during this stage.

``std::vector<ModBlockDataType> m_types;``
	``ModBlockDataType`` is an enum that declares whether or not it is a variable. This is to distinguish between a variable that always uses an ``std::string*``, and a raw value that could use any pointer.

``std::vector<ModBlockDataInterpretation> m_interpretations;``
	``ModBlockDataInterpretation`` is an enum that defines the interpretation of an argument. This is only between preprocessor stages to state how to read and interpret the data inside ``ModBlockData``

TCC, or Tiny C Compiler is not a well-known compiler. Its purpose is to be a "middle man" from the executable and Cappuccino. The PreProcessor (executable side) works with TCC to do a few things. First, it links all mods with ``Cappuccino`` so it has access to the functions inside the mods. Second, the PreProcessor defines these symbols though TCC:

.. code:: c++

	uint64_t functionMain;
	uint64_t* functionCallCount;
	uint64_t functionTotalCount;
	void (***calls)(ModBlockPass*);
	ModBlockData** functionData;
	ModBlock*** modBlocks;
	bool debugBuild;
	uint8_t* superInstruction;
	int64_t* superData;
	void* superMutex;

One symbol you may recognize is ``ModBlockData** functionData``. This contains every blocks' data through the ``ModBlockData``. A few other easily understandable symbols are the ``bool debugBuild`` and ``ModBlock*** modBlocks``. The ``ModBlock***`` is understood as a 2d array of ``ModBlock*``.

``uint64_t functionMain`` contains what stack is marked as an entry (``vin_main``). ``uint64_t functionTotalCount`` denotes how many stacks there are and ``uint64_t* functionCallCount`` is an array. Each element defines the block count in a stack, and each stack has an element.

``void (***calls)(ModBlockPass*)`` looks very confusing, especially if you don't know C. This is basically a pointer to a function that can be called. ``void`` is the return specifier, ``ModBlockPass*`` is the parameter, ``calls`` is the name, and ``***`` part is a multi-pointer. This is a 2d array of function pointers.

``uint8_t* superInstruction`` is an unsigned 8-byte integer that specifies the instruction that can be read and written to by the executable and **Cappuccino**. This is mutexed on both sides though a ``std::mutex*``, disguised as ``void* superMutex``. This is so **TCC** can compile it, but **Registration** (**Cappuccino** side) is fully aware of this being an ``std::mutex*``. Similarly the ``int64_t* superData`` carries the data next to the instruction.

Configurations
--------------

Two very important features of **Unnamed Blocks** are the debug and release build configurations. The debug configuration provides buffers and facilitates identification of issues at the cost of performance. It also allows the use of thread breaking and stepping. The debug configuration is only available with the editor attached.

The release configuration will take longer to compile, and is much less safe and prone to overflows and crashes. The release configuration does anything possible to maximize speed during runtime. It's much faster than the debug configuration because of all the optimizations it puts in place, and minimizes safety guards.

R & L Values
------------

Every **L** value is stored in a text registry as:

.. code:: c++

	("_L_" + data[b])

``data`` is the following member snippet from a ``ModBlockData``:

.. code:: c++

	std::vector<void*> m_data;

This is the data set by the PreProcessor. If this ``ModBlockData``'s ``ModBlockDataType`` is flagged as ``ModBlockDataType::VAR``, then ``data[b]`` will *always* be an ``std::string*``. This is why all **L** values follow this variable-name convention.

**R** values are a little different. To the user they are not variables, they are simpler and *must be faster*. This is not true because **R** values need to be stored somewhere. In terms of runtime performance, they are the same. If you have an excessive quantity of **R** values (i.e., hundreds,) you will increase your compile time and thread-summon time respectively. This is because every single **R** value is stored just like a variable. The following is the **R** value convention:

.. code:: c++

	char buffer[20];
	sprintf(buffer, "_R_%u_%u_%u", i, a, b);

**I** is the stack index, **A** is the block index, and **B** is the argument index. It must be this protected because one block may have more than one **R** value.

Compile Debug
-------------

This is continued from R & L Values. The debug variable registry starts as one text channel. Every time an **R** or **L** value needs to be registered, regardless of the ``ModBlockDataInterpretation``, its text name will be added to this single channel. The ``size()`` of the channel before addition will be the argument's relative index. To add to the registry, it calls a lambda with the following declaration:

.. code:: c++

	[&](const std::string& name, const uint64_t& idx, const ModBlockDataInterpretation& interp, void* use = nullptr)

As you can see, the lambda takes in ``void* use = nullptr``. This parameter is optional (defaulted to ``nullptr``). **R** values use this to initialize the memory. Because all **R** values come with either a ``double*``, ``bool*``, or ``std::string*`` we can initialize it through this function.

.. code:: c++

	addToRegistry(std::string(buffer), i, ModBlockDataInterpretation::REAL, new double(*(double*)data[b]));

Meanwhile **L** values are left as ``nullptr``

.. code:: c++

	addToRegistry("_L_" + *(std::string*)data[b], i, interpretations[b]);

Compile Release
---------------

This is continued from R & L Values and is in response to Compile Debug. The release variable registry minimizes memory by interlacing variables with the same indices, but separated into different channels. This means that if a ``ModBlock`` calls ``GetReal(1)``, but the second parameter is a boolean, then it will return an invalid ``double&``. This could further cause a crash or cause other ``ModBlock``s to corrupt data or files.

ModBlockPass - Variables
------------------------

The ``ModBlockPass`` is passed to a ``ModBlock`` call function pointer. The ``ModBlockPass`` consists of several important features. This page will only be going over how it handles data and requests.

To optimize runtime performance, the ``ModBlockPass`` has the following convention for most use calls:

.. code:: c++

	public:
		double& GetReal(const uint64_t& idx);
	private:
		double& (ModBlockPass::* m_getReal)(const uint64_t& idx);
		double& GetRealDebug(const uint64_t& idx);
		double& GetRealRelease(const uint64_t& idx);

When the user runs ``GetReal(0)``, all it does is call and return a function pointer.

.. code:: c++

	double& ModBlockPass::GetReal(const uint64_t& idx)
	{
		return (this->*(m_getReal))(idx);
	}

This is so the function pointer that is being called can be set to any function. It is useful to be able to change between debug and release configurations. This can be seen in the ``ModBlockPass`` constructor.

.. code:: c++

	if (init.DebugMode)
		m_getReal = &ModBlockPass::GetRealDebug;
		// continued
	else
		m_getReal = &ModBlockPass::GetRealRelease;
		// continued

During a ``GetReal(0)`` call in debug mode, there are several steps.

.. code:: c++

	if (idx >= m_variablesBoolCount->at(m_callstackStackIdx->back()))
	{
		LogError("attempted to get bool out of range \"" + std::to_string(idx) + "\". registry size is \"" + 	std::to_string(m_variablesBoolCount->at(m_callstackStackIdx->back())) + "\"", LoggerFatality::ABORT);
		return gBool;
	}

	const uint64_t& vIdx = m_activeIdx[m_callstackBlockIdx->back()][idx];
	double& value = m_activeReal[vIdx];
	return value;

It first checks to see if the ``idx`` provided is higher than the argument count of the current block. This is only important when developing a mod. It then finds the ``vIdx``, which is the variable index of the active variable stack. Using ``vIdx``, it finds and returns a ``double&``. The release function is the same, except more compact and without bounds checking.

.. code:: c++

	return m_activeReal[m_activeIdx[m_callstackBlockIdx->back()][idx]];

ModBlockPass - Custom
---------------------

Custom data can be altered in three ways to the user:

.. code:: c++

	const uint64_t CustomPut(void* mem);
	void* CustomGet(const uint64_t& idx);
	void CustomFree(const uint64_t& idx, bool deallocate = true);

Unlike variables, customs do not have separate debug and release configurations. The custom registry is shared among all of **Cappuccino**. When ``CustomPut(mem)`` is called, it simply pushes to the registry and returns its index.

.. code:: c++

	std::unique_lock<std::mutex> lock(*m_customRegistrerMutex);

	uint64_t customIdx = m_customRegister->size();
	m_customRegister->push_back(mem);

	return customIdx;

This however is not a good system because this vector's size is never decreased. You can retrieve the memory by using ``CustomGet(idx)``.

.. code:: c++

	std::unique_lock<std::mutex> lock(*m_customRegistrerMutex);
	return m_customRegister->at(idx);

The user should NOT free the memory unless ``CustomFree(idx, false)`` is called with ``false``. This tells **Cappuccino** that the memory is no longer in use and to not free it later. If ``CustomFree(idx, true)`` is called with ``true``, then **Cappuccino** knows that the memory is no longer in use, will not free it at the end, but will free it immediately upon ``CustomFree(idx, true)``.

.. code:: c++

	std::unique_lock<std::mutex> lock(*m_customRegistrerMutex);

	if (deallocate)
		delete m_customRegister->at(idx);

	m_customRegister->at(idx) = nullptr;

ModBlockPass - Callstack
------------------------

This is the convention for all three variable types:

.. code:: c++

	std::vector<double*> m_stackingReal;
	double* m_activeReal;

	std::vector<double*> m_dataStackReal;
	const std::vector<uint64_t>* m_variablesRealCount;

I have deliberately separated the four members into groups of two. These members may look confusing because there are four names that are very similar. I will be breaking down what each of these members do and how they are used throughout the ``ModBlockPass``.

The ``m_variablesRealCount`` may make sense immediately. Every element is the amount of real variables in the corresponding stack. This includes both **R** and **L** Values. This is generally used when checking bounds and allocating the other members listed above.

``m_dataStackReal`` is ground zero. Every element is an array of a variable type for the corresponding stack. This is used as a template to allocate further members. During ``RuntimeInitialization``, ``ModBlock``s will be able to set this default data though the ``ModBlockData``.

.. code:: c++

	const std::vector<void*>& GetData();

**m_stackingReal** and **m_activeReal** are closely related. **m_stackingReal** is simply a "stack" of active variable registries. It grows and shrinks when functions are called. **m_activeReal** simply refers to the most active variable registry.

.. code:: c++
	
	m_activeReal = m_stackingReal.back();
	m_activeBool = m_stackingBool.back();
	m_activeString = m_stackingString.back();

You may be wondering, "Why do you need a member to specify the active stack's variable registry when you can get it using ``m_stackingReal.back()``?"

The answer is: "Performance". A better question would be, "Why can you just set the ``m_activeReal`` to the template registry instead of pushing it to another vector first?" There is a very important reason for this, and it has to do with these public functions:

.. code:: c++

	void AddCallstack(const uint64_t& stack, const uint64_t& block, const bool& special = true);
	void PopCallstack();

If you just wanted to quickly change stacks during runtime, you could do so when **special** were false. This would result in the following code being executed:

.. code:: c++

	m_callstackStackIdx->push_back(stack);
	m_callstackBlockIdx->push_back(block);

	m_stackingSpecial.push_back(false);

	m_stackingReal.push_back(m_dataStackReal[m_callstackStackIdx->back()]);
	m_stackingBool.push_back(m_dataStackBool[m_callstackStackIdx->back()]);
	m_stackingString.push_back(m_dataStackString[m_callstackStackIdx->back()]);

However, you may notice an issue with this approach. Even though the previous index was saved inside ``m_callstackBlockIdx`` and ``m_callstackStackIdx``, the registry that would be set as active would be from the template registry ``m_dataStackReal``. This means that you would be treating all **R** and **L** Values as static. This would mean there could only be one of each in a particular translation unit, or stack in this case.

.. code:: c++

	static double gReal = 0.0;
	static bool gBool = false;
	static std::string gString;


This is a major issue when you are trying to keep multiple states of the same variable in the same stack. When you call the function that you are inside the middle of, the new callstack will be editing the same variables as the old callstack. This can easily cause memory corruption.

So how can we keep multiple states of the same variables? By flagging **special** true. This instead causes the following code to run:

.. code:: c++
	
	m_stackingSpecial.push_back(true);

	double* reals = new double[m_variablesRealCount->at(m_callstackStackIdx->back())];
	bool* bools = new bool[m_variablesBoolCount->at(m_callstackStackIdx->back())];
	std::string* strings = new std::string[m_variablesStringCount->at(m_callstackStackIdx->back())];

	for (uint64_t i = 0; i < m_variablesRealCount->at(m_callstackStackIdx->back()); i++)
		reals[i] = m_dataStackReal[m_callstackStackIdx->back()][i];

	for (uint64_t i = 0; i < m_variablesBoolCount->at(m_callstackStackIdx->back()); i++)
		bools[i] = m_dataStackBool[m_callstackStackIdx->back()][i];

	for (uint64_t i = 0; i < m_variablesStringCount->at(m_callstackStackIdx->back()); i++)
		strings[i] = m_dataStackString[m_callstackStackIdx->back()][i];

	m_stackingReal.push_back(reals);
	m_stackingBool.push_back(bools);
	m_stackingString.push_back(strings);

When **special** is flagged true, instead of pushing the ``m_dataStackReal`` (template registry), it instead makes a copy of it. This way, we can have multiple states of the same variable. When a function comes to the end, it pops the last ``m_stackingReal``.

.. code:: c++

	m_callstackStackIdx->pop_back();
	m_callstackBlockIdx->pop_back();

	if (m_stackingSpecial.back())
	{
		delete[] m_stackingReal.back();
		delete[] m_stackingBool.back();
		delete[] m_stackingString.back();
	}

	m_stackingReal.pop_back();
	m_stackingBool.pop_back();
	m_stackingString.pop_back();

ExecutionThread
---------------

``ExecutionThread`` refers to the thread where the execution is taking place. The ``ExecutionThread`` has several flags:

.. code:: c++

	std::atomic<bool> m_finished;
	std::atomic<bool> m_kill;
	std::atomic<bool> m_ended;

	std::atomic<bool> m_breaked;
	std::atomic<bool>* m_resume;
	std::atomic<bool> m_step;

Due to performance, we don't want the thread checking each thread with every block execution. Similarly, we also do not want to calculate the time since the last flag pull to pull after the time is up. Instead we identified one single flag to be responsible for interrupting the execution and to then test the other flags. ``m_finished`` does exactly this.

As soon as the execution is interrupted, it first tests ``m_breaked``. If this is true, then ``m_finished`` will continue to be false so we perform step execution. The following is a *heavily modified snippet* of what happens when ``m_breaked`` is true:

.. code:: c++

	finished = false;

	while (!resume)
	{
		// true when Cappuccino wants
		// this thread to die immediately
		if (kill)
			return;

		if (step)
		{
			step = false;
			finished = true;
			break;
		}
	}

	if (resume)
		break = false;

	// inside execution loop after
	// m_finished is checked
	goto loop;

Super Instructions
------------------

Super instructions are instructions that can be sent and received from either ``Cappuccino`` or the executable to perform debugging operations. Here are the instructions at the time of writing:

.. code:: c++
	
	1 - [ R/D ] stop; kill all
	2 - [ R/D ] break all
	3 - [ R/D ] resume all
	4 - [ __D ] break single thread (idx)
	5 - [ __D ] step single thread
	6 - [ __D ] resume single thread