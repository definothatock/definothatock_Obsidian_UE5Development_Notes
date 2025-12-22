# The Crashing incident 
I wrote in DC:
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

# Widget Auth check fail
AI replied:
#### Why it fails for the Host

When the Host clicks the button, they are interacting with their local Widget. The `Has Authority` check is being run against their local Player Controller _reference_. While the Host's controller does have authority, relying on actor authority checks inside a Widget Blueprint is notoriously unreliable and bad practice because Widgets are not replicated actors.

# MP in Cooked Package and PIE (related to NM_Client)
https://poe.com/s/A0fF8EGlEYn1HzcbogL7

# Bad UI practice: Control stuck
https://poe.com/s/qEdx7OuxECDlsjkKbToO