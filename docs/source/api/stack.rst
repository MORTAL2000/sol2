stack namespace
===============
the nitty-gritty core abstraction layer over Lua
------------------------------------------------

.. code-block:: cpp

	namespace stack

If you find that the higher level abstractions are not meeting your needs, you may want to delve into the ``stack`` namespace to try and get more out of Sol. ``stack.hpp`` and the ``stack`` namespace define several utilities to work with Lua, including pushing / popping utilities, getters, type checkers, Lua call helpers and more. This namespace is not thoroughly documented as the majority of its interface is mercurial and subject to change between releases to either heavily boost performance or improve the Sol :doc:`api<top>`.

There are, however, a few :ref:`template customization points<extension_points>` that you may use for your purposes and a handful of potentially handy functions. These may help if you're trying to slim down the code you have to write, or if you want to make your types behave differently throughout the Sol stack. Note that overriding the defaults **can** throw out many of the safety guarantees Sol provides: therefore, modify the :ref:`extension points<extension_points>` at your own discretion.

functions
---------

.. code-block:: cpp
	:caption: function: get
	:name: stack-get

	template <typename T>
	auto get( lua_State* L, int index = -1 )

Retrieves the value of the object at ``index`` in the stack. The return type varies based on ``T``: with primitive types, it is usually ``T``: for all unrecognized ``T``, it is generally a ``T&`` or whatever the extension point :ref:`stack::getter\<T><getter>` implementation returns. The type ``T`` has top-level ``const`` qualifiers and reference modifiers removed before being forwarded to the extension point :ref:`stack::getter\<T><getter>` struct.

.. code-block:: cpp
	:caption: function: check

	template <typename T>
	bool check( lua_State* L, int index = -1 )

	template <typename T, typename Handler>
	bool check( lua_State* L, int index, Handler&& handler )

Checks if the object at ``index`` is of type ``T``. If it is not, it will call the ``handler`` function with the index, expected, and actual types.

.. code-block:: cpp
	:caption: function: push

	// push T inferred from call site, pass args... through to extension point
	template <typename T, typename... Args>
	int push( lua_State* L, T&& item, Args&&... args )

	// push T that is explicitly specified, pass args... through to extension point
	template <typename T, typename Arg, typename... Args>
	int push( lua_State* L, Arg&& arg, Args&&... args )

	// recursively call the the above "push" with T inferred, one for each argument
	template <typename... Args>
	int multi_push( lua_State* L, Args&&... args )

Based on how it is called, pushes a variable amount of objects onto the stack. in 99% of cases, returns for 1 object pushed onto the stack. For the case of a ``std::tuple<...>``, it recursively pushes each object contained inside the tuple, from left to right, resulting in a variable number of things pushed onto the stack (this enables multi-valued returns when binding a C++ function to a Lua). Can be called with ``sol::stack::push<T>( L, args... )`` to have arguments different from the type that wants to be pushed, or ``sol::stack::push( L, arg, args... )`` where ``T`` will be inferred from ``arg``. The final form of this function is ``sol::stack::multi_push``, which will call one ``sol::stack::push`` for each argument. The ``T`` that describes what to push is first sanitized by removing top-level ``const`` qualifiers and reference qualifiers before being forwarded to the extension point :ref:`stack::pusher\<T><pusher>` struct.

.. code-block:: cpp
	:caption: function: set_field

	template <bool global = false, typename Key, typename Value>
	void set_field( lua_State* L, Key&& k, Value&& v );

	template <bool global = false, typename Key, typename Value>
	void set_field( lua_State* L, Key&& k, Value&& v, int objectindex);

Sets the field referenced by the key ``k`` to the given value ``v``, by pushing the key onto the stack, pushing the value onto the stack, and then doing the equivalent of ``lua_setfield`` for the object at the given ``objectindex``. Performs optimizations and calls faster verions of the function if the type of ``Key`` is considered a c-style string and/or if its also marked by the templated ``global`` argument to be a global.

.. code-block:: cpp
	:caption: function: get_field

	template <bool global = false, typename Key>
	void get_field( lua_State* L, Key&& k [, int objectindex] );

Gets the field referenced by the key ``k``, by pushing the key onto the stack, and then doing the equivalent of ``lua_getfield``. Performs optimizations and calls faster verions of the function if the type of ``Key`` is considered a c-style string and/or if its also marked by the templated ``global`` argument to be a global.

This function leaves the retrieved value on the stack.

.. _extension_points:

objects (extension points)
--------------------------

The structs below are already overriden for a handful of types. If you try to mess with them for the types ``sol`` has already overriden them for, you're in for a world of thick template error traces and headaches. Overriding them for your own user defined types should be just fine, however.

.. _getter:

.. code-block:: cpp
	:caption: struct: getter

	template <typename T, typename = void>
	struct getter {
		static T get (int index = -1) {
			// ...
			return // T, or something related to T.
		}
	};

This is an SFINAE-friendly struct that is meant to expose static function ``get`` that returns a ``T``, or something convertible to it. The default implementation assumes ``T`` is a usertype and pulls out a userdata from Lua before attempting to cast it to the desired ``T``. There are implementations for getting numbers (``std::is_floating``, ``std::is_integral``-matching types), getting ``std::string`` and ``const char*``, getting raw userdata with :doc:`userdata_value<types>` and anything as upvalues with :doc:`upvalue_index<types>`, getting raw `lua_CFunction`_ s, and finally pulling out Lua functions into ``std::function<R(Args...)>``. It is also defined for anything that derives from :doc:`sol::reference<reference>`.

.. _pusher:

.. code-block:: cpp
	:caption: struct: pusher

	template <typename X, typename = void>
	struct pusher {
		template <typename T> 
		static int push ( lua_State* L, T&&, ... ) {
			// can optionally take more than just 1 argument
			// ...
			return // number of things pushed onto the stack
		}
	};

This is an SFINAE-friendly struct that is meant to expose static function ``push`` that returns the number of things pushed onto the stack. The default implementation assumes ``T`` is a usertype and pushes a userdata into Lua with a :doc:`usertype_traits\<T><usertype>` metatable associated with it. There are implementations for pushing numbers (``std::is_floating``, ``std::is_integral``-matching types), getting ``std::string`` and ``const char*``, getting raw userdata with :doc:`userdata<types>` and raw upvalues with :doc:`upvalue<types>`, getting raw `lua_CFunction`_ s, and finally pulling out Lua functions into ``sol::function``. It is also defined for anything that derives from :doc:`sol::reference<reference>`.

.. code-block:: cpp
	:caption: struct: checker

	template <typename T, type expected = lua_type_of<T>, typename = void>
	struct checker {
		template <typename Handler>
		static bool check ( lua_State* L, int index, Handler&& handler ) {
			// if the object in the Lua stack at index is a T, return true
			if ( ... ) return true;
			// otherwise, call the handler function,
			// with the required 4 arguments, then return false
			handler(L, index, expected, indextype);
            return false;
		}
	};

This is an SFINAE-friendly struct that is meant to expose static function ``check`` that returns the number of things pushed onto the stack. The default implementation simply checks whether the expected type passed in through the template is equal to the type of the object at the specified index in the Lua stack. The default implementation for types which are considered ``userdata`` go through a myriad of checks to support checking if a type is *actually* of type ``T`` or if its the base class of what it actually stored as a userdata in that index.

.. _lua_CFunction: http://www.Lua.org/manual/5.3/manual.html#lua_CFunction