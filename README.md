collector
=========

A concurrent garbage collector for C++. Call it periodically from a background thread to sweep up your junk.

### Advangates

* Unlike reference counting, handles cycles.
* Usable in a real-time thread, because collection can occur in another thread. (Just need to collect often enough, see below)
* Supports multiple mutator threads. ("Mutator" threads are just your threads that aren't the collector thread.)
* Coexists peacfully with other forms of C++ memory management.
* Defers collection until you want to do it.
* Offers the same conncurrency guarantees as `shared_ptr` (I think, hah)
* Battle-tested in a real app (I haven't attributed any bugs to the collector, but I make no guarantees!)
* < 500 lines and dirt simple.

### Disadvantages

* The author is a noob. He's just a graphics programmer with no real experience with GC.
* Intrusive. You need to derive from `Collectable`. Has to be intrusive so virtual destructors can be called from the collector.
* Requires a little discipline with use of `RootPtr` and `EdgePtr` smart pointers.
* Pointer assignment is probably super slow because it puts an event on a queue (I haven't profiled it).
* Currently, if you don't collect often enough, a fixed-size queue will fill up and you'll block.
* Collection isn't optimized.
* Your code can block if you don't collect often enough.

### How to Use

You'll need [Boost](http://www.boost.org) and C++11. Drop `Collector.cpp` and `Collector.hpp` in your project. 

Let's say you're doing a graph data structure. You might have something like this:

```c++
class Node : public Collectable {
  
 public:
  
  // Pass a pointer to a Node via a RootPtr, since it's on the stack.
  void AddEdge(const RootPtr<Node> &node) {
  
    // Store pointers in the heap as EdgePtrs.
    _edges.push_back( EdgePtr<Node>(this, node) );
  }
  
 private:

  std::vector< EdgePtr< Node > > _edges;
};
```

The basic idea is whenever you are pointing to a `Collectable` on the stack, use a `RootPtr`. For pointers living in the heap, use an `EdgePtr`. (Maybe they should be called `StackPtr` and `HeapPtr`. I just use terms from the GC research papers I can't understand.)

To create a collected object in the example, we'd do:

```c++
RootPtr<Node> node(new Node); // The collector uses the normal new and delete allocators.
```

There are two constructors for `EdgePtr`:

* `EdgePtr<T>::EdgePtr(Collector* owner)` Creates a NULL `EdgePtr` with an owner.
* `EdgePtr<T>::EdgePtr(Collector* owner, Collector* p)` Creates an `EdgePtr` from `owner` to `p`.

You can shoot yourself in the foot by forgetting about the `owner`. In practice I've found it pretty easy to avoid doing so.

When you want to collect, just call: `Collector::GetInstance().Collect()` periodically from a background thread. If you only call it from the main thread, your code can deadlock. To reduce blocking, you can call `Collector::GetInstance().ProcessEvents()` in the background thread more often than you call `Collect`. My collector thread looks like this:

```c++
	while (go) {

		// Update frequently to keep the event
		// queue from filling up.
		Collector::GetInstance().ProcessEvents();

		// Collect occasionally.
		if (n % 100) {
			Collector::GetInstance().Collect();
		}

		// Sleep for .01 seconds.
		boost::this_thread::sleep(boost::posix_time::millisec(10));

	}
```

Enjoy!

### Todo
* Reduce the possibility that the mutator thread will block
* Add some debug sanity checks for `RootPtr` and `EdgePtr`.
* Be less of a noob.