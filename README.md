# stop thinking
It was for OS development purposes, but I was pushed by the lightness of the C language, and I gave up because the memory waste from abstraction was too big.
Turn to development at the application level.  However, functions, etc., use system calls. The basic concept is to be taken.

# bocchi-the-programming
bocchi the programming! - new language project

# syntax example
```
// load macro (compile time processing, but also structure info will load memory for dynamic resolving)
#define_structure person {
    string name;
    int age;
};

#load_structure("test_structure.structure");


const system = import_value("system", "1.0.0"); // system info dynamic only (not compile time)
const open_file = import_function("file", "open_file", "1.0.0");
const open_file_dynamic = import_function("file", "open_file", "1.0.0"); // dynamic linking (without '#')
const load_text_from_file = import_function("file", "load_text_from_file", "1.0.0");
const load_json_from_file = import_function("file", "load_json_from_file", "1.0.0");
const load_binary_from_file = import_function("file", "load_binary_from_file", "1.0.0");
const sqrt = import_function("math", "sqrt", "1.0.0");
const function print = import_function("console", "print", "1.0.0");
const math_value = import_value("math", "1.0.0");
const default_value = import_value("default", "1.0.0");
const environment_variables = import_value("environment", "1.0.0"); // environment variable dynamic only (not compile time)

function add(a, int32 b) { // you can specific type of argument
    return a + b;
}

function greet(name) {
    return f"Hello, {name}!";
}

function get_binary() {
    return b"ff010203";
}

function get_all_args(all_args list) { // you can get all args by list
    for(arg in list) {
        print(arg)
    }
}

// Automatically progress the garbage collection at the end of the loop.
function main_loop() {
    global sqrt;
    global print;
    global math_value;
    global default_value;

    person bocchi; // same as 'list bocchi = list(2)'
    bocchi@name = "Guitar"; // same as 'bocchi[1] = "Guitar"'
    print(bocchi@name); // same as 'print(bocchi[1])'

    // inline assembly
    __asm__("mov cr0, eax");

    // resource deallocation when out of scope.
    with(resource file = open_file("readme.txt")) {
        string txt = load_text_from_file(file);
        print(txt);
    }

    print(math_value.pi); // 3.14...

    map name_age = {"name": "Alice", "age": 30}; // map example
    print(name_age.name); // print Alice
    print(name_age["name"]); // print Alice

    print(get_keys(name_age)); // list : ["name", "age"]
    print(get_value(name_age)); // list : ["Alice", 30]

    string test_string = "12345";
    print(get_allocation_length(test_string)); // print 32 bytes (2^5) : capacity
    print(get_byte_length(test_string)); // print 20 bytes (UTF-32)
    print(get_string_length(test_string)); // print 5
    print(test_string[2]); // print 3

    string cloned_string = clone(test_string); // clone copy (1 depth)
    if(get_type(cloned_string) === default_value.type_string) {
        print(cloned_string);
    }

    if(get_memory_location(cloned_string) === default_value.memory_temporary) { // GC target memory. (in case of 'memory_global', not GC target)
        print(cloned_string);
    }

    compare_memory(test_string, cloned_string); // same memcmp

    items = list(32); // capacity
    items.append(123);
    items.append("hello");
    items.append(true);

    set_example = set(32); // capacity
    set_example.add(123);
    set_example.add("hello");
    set_example.add(true);

    map_example = map(16); // capacity
    map_example.put("hello", 123);
    map_example.put("test", true);
    print(map_example["hello"]) // print 123

    function add2 = register_function("application", "add2", "1.0.0", "function add2(a, b){ return a + b; }", system@architecture); // application library only for use internal. architecture for compile

    var num1 = sqrt(100);
    var num2 = 20;

    var result1 = add(num1, num2);
    print("The sum is: " + result1);
    var result2 = add2(num1, num2);
    print("The imported sum is: " + result2);

    unregister_function("application", "add2", "1.0.0", system@architecture); // remove function

    print(greet("Bocchi"));
    
    var global_string = allocate_global("I'm a static variable."); // global variable allocation
    free_global(global_string);

    dump_memory(); // display all memory

    return false; // If return true, It continues loop. otherwise, end of loop.
}
```

# concept
- having a grammar similar to javascript / python
- However, the system is programmable, so the OS can be developed
- Prioritizes time complexity over space complexity
- Strength in string processing.
- Adopts a value-centric memory model even at the cost of some overhead.
- Value structure (pseudo written by C)
```c
struct Value {
    void* ptr;           // Pointer to the data or function code
    void* ptr2;          // Auxiliary pointer (e.g., for map key arrays, closure captures, etc.), also use for resource deallocator, also use for hash (map for string)

    uint64_t type;        // Type of the value (e.g., int8, uint32, list, map, string, function, resource, etc.)
    uint64_t memory_location;      // Memory location (STACK, GC_HEAP, GLOBAL_HEAP, STATIC, CODE)
    uint64_t is_refernce; // Reference flag (1 if it's a reference and should not be deallocated)

    uint64_t length;     // Logical length (e.g., for lists/strings), or the actual value for integers
    uint64_t capacity;   // Capacity for array-like structures
};
```
- Memory Area
  - STACK: Stack frames for local variables and function calls
  - GC_HEAP: Heap managed by GC on a per-main_loop basis
  - GLOBAL_HEAP: Global heap, persists independently of the loop
  - STATIC: Fixed resources that require explicit deallocation
  - CODE_EXEC: Executable code area (populated after user code is compiled)
  - CODE_LIBRARY: Code space for dynamically loadable/unloadable library functions
- This structure enables real-time monitoring of memory allocation status.
- UTF-32 is used to prioritize efficiency in string processing.
- Functions are version-controlled individually by default, and the CODE_LIBRARY area is designed to support downloading from the internet in the future.
- Specifies the functions and value required in the executable file (metadata). Loads the functions and value required before running the program.
- Provides built-in linear memory structures: dequeue, list, map, and set.
- Defaults to a 64-bit architecture.
- The language includes its own compiler, allowing dynamic compilation and library registration. (experimental)
```
  register_function("math", "sqrt", "1.0.0", "function add(a, b){ return a + b;}")
```
- Primitive data types (all types are 'Value' structure)
  - bool
  - int8,  uint8
  - int16, uint16
  - int32, uint32
  - int64, uint64
  - float32, float64
  - char  # UTF-32 (4byte)
  - string  # UTF-32, not null-terminated
  - function
  - map
  - list
  - set
  - resource // for file, socket, etc
- Pseudo code for language flow
- Compiler has these options:
  - "--strict": when finish program, it checks unfreed global variable and print memory leak errors.
  - "--register-function=math@sqrt@1.0.0": add function into library. (not generate main_loop)
  - "--no-main-loop": do not make main_loop. use main instead. (for making OS, driver, etc...)
  - "--expose-gc": expose garbage collection function gc()
```
[Program Start]
→ Load static/global resources
→ while (true):
      - is_continue = main_loop() // Execute main loop
      - gc() // Clear temp heap (GC heap), also auto deallocation GC_HEAP 'resource'
      - if(!is_continue) break
→ Check unfreed static memory
→ Exit
```
