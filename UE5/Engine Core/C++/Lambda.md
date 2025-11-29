A lambda (since C++11) is an **inline, unnamed function object** you create right where you need it. It's perfect for short tasks—particularly when passing behavior to algorithms (like `std::sort`) or asynchronous code.

Example from Project:

	void UWorkingMemoryArrayCortex::RemoveExpired()
	{
		Entries.RemoveAllSwap(
		[](const FWorkingMemoryEntry& Entry)
			{
				return (Entry.TTL <= EXPIRE_TRUSTHOLD || !Entry.IsValid());
			});

⚠️ Be careful: capturing by reference can dangle if the lambda outlives the referenced object.
