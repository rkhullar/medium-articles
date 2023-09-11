all: clean setup style readme.html post

clean:
	@rm -rf dist

post:
	@rm dist/header.html

setup:
	@mkdir -p dist
	@ln -s `pwd`/images dist/images

style:
	@pygmentize -S default -f html -a .codehilite > dist/codehilite.css
	@echo '<link rel="stylesheet" href="codehilite.css"/>' > dist/header.html

%.html: %.md
	@mkdir -p dist
	@cat $< > dist/$@-build.md
	@cat dist/$@-build.md | python -m markdown -x fenced_code -x codehilite > dist/$@.body
	@cat dist/header.html dist/$@.body > dist/$@
	@rm dist/$@.body dist/$@-build.md

%.md:
