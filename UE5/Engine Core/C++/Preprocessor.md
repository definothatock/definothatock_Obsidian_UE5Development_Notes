A stage that runs before actual compilation.

Example:

		#ifdef DEBUG
		#define LOG(msg) std::cerr << msg << '\n'
		#else
		#define LOG(msg) ((void)0)
		#endif

- `#ifdef DEBUG` checks whether a macro named `DEBUG` has been defined (using `#define DEBUG` or a compiler flag like `-DDEBUG`).
- If `DEBUG` **is** defined, the macro `LOG(msg)` expands to `std::cerr << msg << '\n'`, i.e., it prints the message to standard error.
- If `DEBUG` **is not** defined, `LOG(msg)` expands to `((void)0)`, which effectively does nothing.
- `#endif` closes the conditional.

To define DEBUG from the above example, we can either:
- In code
	`#define DEBUG`
- In compiler
	`g++ -DDEBUG main.cpp -o main`


This pattern keeps logging statements throughout code without paying any runtime cost in release builds.


[[C++]]