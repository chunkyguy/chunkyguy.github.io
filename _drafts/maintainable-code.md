# Break down into components

- Data and functions. Every function cares only about a subset of data.
- List component only deals with dates for sorting
- Render component only cares about rendering text.
    - Can be extended to support images, links, ...
- DiskIO component only cares about encoded data
