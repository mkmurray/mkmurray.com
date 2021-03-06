---
layout: post
title: "Refactoring Legacy Code Starts with Test Coverage"
date: 2012-11-10 15:37
comments: true
categories:
 - Legacy Code
 - Refactoring
 - Ruby
 - Testing
---

As a dev team, we have been watching and discussing various training videos and
conference talks every morning for a few months now. This week we watched a
recorded conference talk by Ruby developer [Katrina Owen](http://kytrinyx.com/)
entitled [Therapeutic Refactoring](http://www.youtube.com/watch?v=J4dlF0kcThQ).
I was especially impressed with her strategy for refactoring legacy code.

I'm sure everyone can agree that the safest way to refactor code and be 100%
positive that the functionality hasn't regressed is to have a good suite of
tests. And if you don't have sufficient test coverage (or any at all), then now
is the right time to add it before refactoring.

However, I have found myself skipping this step in the past, feeling justified
because the quality of the code was so poor or the deadline was so short. It is
irresponsible of me to assume that I have not changed the functionality of the
code without proof in the form of passing tests that sufficiently cover all
scenarios and use cases.  Nonetheless, watching Katrina's presentation has
given me renewed commitment and a sound plan of attack against legacy code that
may be ugly and very unfamiliar to me.

Here is the example method that Katrina uses during her presentation (which is
later revealed as a piece of code that she vaguely remembers coding herself and
now abhors):
<pre class="brush: ruby">
    module XYZService

      def self.xyz_filename(target)
        # File format:
        # [day of month zero-padded][three-letter prefix] \
        # _[kind]_[age_if_kind_personal]_[target.id] \
        # _[8 random chars]_[10 first chars of title].jpg
        filename = "#{target.publish_on.strftime("%d")}"
        filename << "#{target.xyz_category_prefix}"
        filename << "#{target.kind.gsub("_", "")}"
        filename << "_%03d" % (target.age || 0) if target.personal?
        filename << "_#{target.id.to_s}"
        filename << "_#{Digest::SHA1.hexdigest(rand(10000).to_s)[0,8]}"
        truncated_title = target.title.gsub(/[^\[a-z\]]/i, '').downcase
        truncate_to = truncated_title.length > 9 ? 9 : truncated_title.length
        filename << "_#{truncated_title[0..(truncate_to)]}"
        filename << ".jpg"
        return filename
      end

    end
</pre>

It certainly isn't the worst lines of code ever written and it is getting the
job done, but it is definitely not very readable and quite difficult to
understand its purpose and logic. About the only thing that can be discerned is
that it is intended to generate a filename in a specific format.

Katrina starts by writing a very simple and seemingly useless test (which she
calls a "Mickey Mouse test" in her GitHub commit message) in order to begin test
coverage of this method:
<pre class="brush: ruby">
    require_relative './xyz_service'

    describe XYZService do

      let(:target) do
        stub(:target)
      end

      subject { XYZService.xyz_filename(target) }

      it 'works' do
        subject.should eq('something')
      end

    end
</pre>

Without much knowledge of what this method's purpose or inputs are, we can only
pass in an empty stub object as input and then assert something rediculous for
the result. We expect this test to not only fail to pass, but it is also likely
to error during runtime (or even at compile time if in a statically typed
language like C#). The idea is to create a quick feedback loop that aids us in
finding the context and inputs required to make the code execute without runtime
error. Then we can finally see the actual return value and modify our dummy
assertion to begin expecting that output.

After iterating in this fashion, Katrina arrives at the following test which is
a bit more useful:
<pre class="brush: ruby">
    require_relative './xyz_service'
    require 'date'

    describe XYZService do

      let(:target) do
        messages = {
          :publish_on => Date.new(2012, 3, 14),
          :xyz_category_prefix => 'abc',
          :kind => 'unicorn',
          :personal? => false,
          :id => 1337,
          :title => 'magic & superglue'
        }
        stub(:target, messages)
      end

      subject { XYZService.xyz_filename(target) }

      it 'works' do
        subject.should eq('14abcunicorn_1337_cb6c53bc_magicsuper.jpg')
      end

    end
</pre>

However, this test doesn't pass because the `cb6c53bc` portion of the resultant
filename is different every execution of the test. A simple regex allows the
test to pass all of the time:
<pre class="brush: ruby">
    subject.should match(/14abcunicorn_1337_[0-9a-f]{8}_magicsuper\.jpg/)
</pre>

Even though we now have a passing test, we have not yet exercised the different
input values and all code paths and branches through the method. This must be
done in order to fully understand, document, and test cover the logic and
purpose of the method. This would be done in the normal unit testing fashion
that is shown in just about every testing demo by adding more test scenarios and
assertions that excercise the varying use cases.

One interesting development that occurred while Katrina was using this process
was that she actually found a bug in the implementation of the original
`xyz_filename` method, specifically an extra set of braces in the regex on line
14.

Now that we are fully covered with tests (and a bug fixed), we are free to
refactor the code into smaller, more readable chunks of code logic that are more
cohesive and more reusable as well. This is what Katrina does for the remainder
of her presentation, and it is just as instructive as this first part. However,
today I wanted to focus on her strategy for writing test coverage against a
legacy codebase.

I like how what Katrina is demonstrating seems to utilize a black box testing
approach at first, and then progresses to a more white box approach as we begin
to explore the different parts of the implementation. Perhaps such a strategy
should have been obvious to me, but Katrina's conference talk really struck a
chord with me and encouraged me to commit to doing better with test coverage in
legacy codebases before I begin making modifications or additions.

I highly recommend viewing the entire video, as it is only 30 minutes long and
an entertaining watch.
