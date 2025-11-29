The Crashing incident 

	ue crashed have to redo pass 3 days' work ! 🥲
	the whole thing is:
	
	- I deleted and moved assets, so in the ctrl z history now exists nullptr
	- I spammed ctrl z because I was experimenting sth and decided whatever I was doing was shit
	- I went past the nullptrs (which the engine did warn me about when I deleted certain assets)
	- not realizing I went into nullptrs, I spammed ctrl y to redo sth
	- engine is confused by the constant nullptr access, it hung
	- I thought its a casual ue lag and opt out for a refresh
	- reopen + restore work (big mistake)
	- crashes
	- now AI cannot access the correct ptrs, and I cannot access the source of trouble because there is no ptr to it
	
	guys please do not do what I done
