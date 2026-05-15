Problem:
1. See other company's data on refresh - Data leak between tenants where one tenants sees the revenue of the other. This has either something to do with how we query database and if we are actually accessing per clientID 

- Check also on API or authentication if there could be an issue of fetching other client details but most likely database query issue


2. March totals wrong - This is a timezone issue, how are we filtering revenue by date and month, need to see how the filter of datetime is applied and used



3. Revenue off by cents - How are we rounding off revenue, if off by cents this is most likely rounding off issue.




- Must check current schema to see the dataset and why its different from what is being presented to the client.


1. Both seed data have same tenant-id, this is one obvious issue that could result to the problem fetching and receiving another clients data, this is also the cause for privacy breach