# Metro Clinic - Web Exploit

While the NCL gym does provide solutions, they tend to jump to conclusions quickly so I wanted to explain my thought process.
I would consider myself a beginner for SQLi, so hopefully this is helpful for anyone in a similar place.

We are provided with a website whose only distinctive feature is a search bar on the main page:
![image](https://user-images.githubusercontent.com/37031448/188216600-be8cd78e-7b94-4969-8523-5dc44fa57aff.png)

I tried searching for a letter first:
![image](https://user-images.githubusercontent.com/37031448/188216896-ae67b0cc-ce8e-46a2-87f4-d7500a8fbe0f.png)

Looks like the input is passed as an URL parameter, and any employeee with an "a" in their name is listed.

Let's look at the CTF questions for guidance here. One of the early questions asks for a total number of employees. 
From this, we can assume that there's some way to bypass the search filter and display all users, not just those with the letter(s) you entered.
This is where the trial and error comes in. Since the parameter is a string, I can guess that the SQL command is likely terminating the input with a `'` or `"`.

No luck with `'`:
![image](https://user-images.githubusercontent.com/37031448/188217858-39495ab2-07e9-4337-a013-21c7ac579d0c.png)

Aha! Two valuable pieces of information - `"` is being used and the database is SQLite:
![image](https://user-images.githubusercontent.com/37031448/188217910-143e1d68-1438-454a-a32b-67247d75608f.png)

From here, we can bypass the search filter. 

Two ways to do it (I'm sure there's others):
- `" or ""="`, `" or "1"="1`, etc - make the WHERE condition always evaluate to True
- `" --` - comment out the rest of the query

Using `" --` (URL encoding makes it look weird):
![image](https://user-images.githubusercontent.com/37031448/188218683-f3b9b14d-ee0e-4db9-827e-e26be84e4107.png)

Bingo! We have a full user listing. This answers the first three CTF questions.
The remaining questions relate to dumping the list of usernames and passwords from the database. 
To retrieve information that isn't in the original query, we can use a UNION-based attack.

First let's see how many columns we can pull down at a time (our UNION query needs to have the same number of columns as the original query).
We try to sort the results by a column number using `ORDER BY` until we get an error:
- `" ORDER BY 1--`	no error!
- `" ORDER BY 2--`	no error!
- `" ORDER BY 3--`	error!
So we need to query two columns.

At this point I referenced the SQLite documentation to figure out where information about tables and their structure is stored (https://www.sqlite.org/schematab.html).
Looks like there's a table called `sqlite_master` that has a `tbl_name` column (name of all tables) and a `sql` column (structure of all tables).

Since we need to query two columns, let's pull down both:

```" union SELECT tbl_name,sql FROM sqlite_master--```
![image](https://user-images.githubusercontent.com/37031448/188220024-81decae8-662a-4272-8713-7367f75906b1.png)


> Note: if you needed to query more columns (for this example, three), you can just pad it with a number

> ```" union SELECT 1,tbl_name,sql FROM sqlite_master--```

> Or just query the same column multiple times

> ```" union SELECT sql,sql,sql FROM sqlite_master--```

Now we have the table structure, let's dump the names and passwords:
`" union SELECT name,password FROM users--`

![image](https://user-images.githubusercontent.com/37031448/188220337-b7957b3c-2a06-4922-9d12-0ed6145d2824.png)

We want usernames too. You could query name/password and then username/password and piece them together.
Or you can use string concatenation to pull them all down at the same time:
`" union SELECT name,username||':'||password FROM users--`

![image](https://user-images.githubusercontent.com/37031448/188220479-f54c07dd-7188-448a-921f-0589bc37e495.png)

That's a wrap, folks.
