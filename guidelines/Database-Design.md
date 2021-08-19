## Design rules

1. Consider Every Viewpoint During Planning  
- Don’t start building a database without input from the product manager and other stakeholders.
- Get consensus on precise expectations, and consider how hard it will be to train users on the search functions.


2. Normalization vs Denormalization  
- With a complicated DB structure, it’s critical to normalize data to achieve minimum redundancy but also required readability.
- Eliminate multi-valued attributes and repeated attributes, then start in on the subkeys.
https://bambielli.com/til/2018-03-03-normalized-vs-denormalized-data/

3. Make Structures Transparent   
The database belongs to its future users, not its creator, so design with them in mind.
- Stay away from shortcuts, abbreviations, or plurals. Use consistent naming conventions.


4. Define Constraints to Maintain Data Integrity  
- FKs


5. Document Everything  
No matter how annoying it may seem, documentation is as essential as primary keys. Take care to document the design, entity-relationship schemas, and triggers for future users.



6. Keep Privacy Primary    
The GDPR signals an era of increasing privacy concerns. Encrypt passwords, and don’t assign an administrator without privacy training and well-documented qualifications.

Vulnerabilities impact data integrity, which impacts everything else in the enterprise.


7. Optimize for Speed   
Create indexes for queries that will be used regularly. Use a database analyzer to determine if an index or a clustered index is necessary.
Consider incorporating tools like Elastisearch to speed up searches.


8. Keep the Database on Separate Server  
Put the database on a different server than the web to lower CPU usage or use managed database.
