**Make container publicly readable**
```bash
swift post -U admin:admin -K admin -r '.r:*' container
```
**Set site index file**
```bash
swift post -U admin:admin -K admin -m 'web-index:index.html' container
```
**Enable file listing**
```bash
swift post -U admin:admin -K admin -m 'web-listings: true' container
```
**Enable CSS for file listing**
