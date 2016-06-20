Prerequisites
========
- Node 4 (Node 6 does not work due to outdated dependencies of hexo-clean-css)
- aws-cli – For deploying

Running locally
=======
npm install
npm run server
Navigate to http://localhost:4000

Deploy
=======
npm install
npm run build
aws s3 sync public/ s3://tech.appear.in --acl public-read
