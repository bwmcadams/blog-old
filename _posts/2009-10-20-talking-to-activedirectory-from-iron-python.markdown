---
title: Talking to ActiveDirectory from IronPython
tags: tech IronPython Silverlight Python ActiveDirectory LDAP
layout: post
---
We're building a new intranet system at work, and I've been toying with a few things that the Windows admin asked for.  Namely, since the secretaries here will update the intranet data to add people's Work & Emergency contact numbers, AIM handles, email addresses, etc. that we find a way to keep it all in sync with ActiveDirectory.  Thereby keeping all the Outlooks  and Blackberries up to date with the latest contact information.

This seemed like a fairly reasonable request, presuming we could figure out how to do it and since I've been using Mono and IronPython a lot more lately, I figured there would be a way to accomplish it.  Most of the information I found online was either really old and/or crappy docs for doing it in C#, or more commonly using PowerShell or VBScript.  So, I managed to poke around and sort out how to get IronPython on Mono (IronPython 2.6RC + Mono 2.4.2.3) to find and update our users.
 
The end result is that I can now, from IronPython, find and update valid information on ActiveDirectory entries to reflect the latest and greatest information.  One thing to note, the MS .Net ActiveDirectory APIs (System.DirectoryServices, which is mirrored in Mono) do something that confused and annoyed me.  There are a limited set of 'valid' attribute keys for a user object in Active Directory (Which is really just LDAP, in case you didn't know).  The `DirectoryEntry` object has a Properties attribute, which contains a hashmap of these values. 

The object will **not** allow you to set an "Invalid" key \(see [this list](http://www.dotnetactivedirectory.com/Understanding_LDAP_Active_Directory_User_Object_Properties.html) for valid keys\).  But if you call `.Properties.Keys` you *only* get back the Properties that have values set.  So, it doesn't appear to be possible to actually ask *What keys are valid?* and do some introspective programming.  I have written a wrapper class to make the DirectoryEntry properties look a bit more pythonic (but disabled support for multi-value attributes for now) - at some point in the near future i'll likely add in a "valid value" filter.

The end result is, if I want to find my own user in ActiveDirectory by my name, I can do the following from the IronPython console:

{% highlight python %}
>>> import ad_util
>>> adh = ad_util.ActiveDirectorySearcher('mydomaincontroller.hostname.or.ip', 'my.domain', 'myUsernameAllowedToChangeObjects', 'myPassword')
>>> adh
<ActiveDirectorySearcher object at 0x000000000000002D>
>>> userObj = adh.find_name('McAdams', 'Brendan')
[debug] Searching activedirectory for (&amp;(objectCategory=user)(objectClass=person)(sAMAccountName=*)(sn=McAdams*)
(givenname=Brendan*)).  Allow multiple results? False
>>> userObj
<DirectoryEntryHelper:{sn=McAdams,givenName=Brendan,mail=None,sAMAccountName=brendan.mcadams}>
{% endhighlight %}

You'll note the object returned is a "DirectoryEntryHelper" type; this is a crappy little wrapper class I put together to simplify attribute access, etc.   You can tell any of the find_ methods to return a raw .Net API object instead of wrapped by passing the kwarg `nowrap=True`;  `find_name()` takes `last_name, first_name`.  Several other utility methods exist on the class including find by account name.  Note that, in order to filter out entries in the Global Address List I'm requiring the search to find objects who have sAMAccountNames. I know the bare minimum about ActiveDirectory, but my Windows admin here tells me that sAMAccountName is a required and *unique* attribute on any actual domain account object.

You can see the keys that are already defined on the object with the `keys` helper method I added onto `DirectoryEntryHelper`:

{% highlight python %}
>>> userObj.keys
['lockouttime', 'primarygroupid', 'msexchuseraccountcontrol', 'distinguishedname', 'cn', 'dscorepropagationdata', 
'whencreated', 'logoncount', 'msexchhomeservername', 'objectclass', 'memberof', 'lastlogontimestamp', 'displayname',    'msexchalobjectversion', 'objectguid', 'whenchanged', 'badpwdcount', 'useraccountcontrol', 'badpasswordtime', 'name', 
'samaccountname', 'mdbusedefaults', 'accountexpires', 'countrycode', 'msds-supportedencryptiontypes', 'homedirectory', 
'userprincipalname', 'lastlogon', 'objectsid', 'givenname', 'homedrive', 'usncreated', 'admincount', 'instancetype', 
'codepage', 'physicaldeliveryofficename', 'samaccounttype', 'sn', 'objectcategory', 'telephonenumber', 'pwdlastset', 
'usnchanged', 'lastlogoff', 'initials']
{% endhighlight %}

This returns a comprehended list of the keys (Rather than calling resultObj.Properties.Keys, which returns a Hashtable of HashKeys.  It just makes life easier.
 
If I want to setup my mobile phone # (which isn't already set already) in Active Directory:

{% highlight python %}
>>> userObj.mobile
# Nothing returned as mobile isn't set...
>>> userObj.mobile = '(646) 555-1212'
# It is committed on set to Active Directory, so another find gets it...
>>> newUserObj = test.adh.find_name('McAdams', 'Brendan')   
[debug] Searching activedirectory for (&amp;(objectCategory=user)(objectClass=person)(sAMAccountName=*)(sn=McAdams*)
(givenname=Brendan*)).  Allow multiple results? False 
>>> newUserObj.mobile
'(646)555-1212'
{% endhighlight %}

I'll leave the rest as an exercise for the reader, but I'm interested in comments and changes.  You can fetch the latest code from [my BitBucket toybox](http://bitbucket.org/bwmcadams/toybox/src/tip/ad_util.py).  I've also pasted it, for posterity sake, below the fold.
 
{% highlight python %}
#!IronPython Specific Script!
# 
# Brendan W. McAdams <bwmcadams@gmail.com>
#
#------------------------------------------------- 
# No copyright or licensing, made public
# as example code.  Feel free to use it as you like;
# No warranty, liability or guarantees are implied.  
# In other words, if you use it YOU ARE ON YOUR OWN. 
#-------------------------------------------------
#
# Utility script to interface from IronPython to 
# MS Active Directory, allowing queries of properties 
# such as telephone numbers, and changes to said properties
# if you'd like to update them.
# 
# Note that your authentication user has to have valid permissions.
# If you're uncertain as to what permissions are needed, please see
# your AD Admin.

import clr 

# Add the System.DirectoryServices Mono DLL in.
# not sure on Windows what you need to add,
# but for Mono/Linux you just need to set IRONPYTHONPATH
# to include the location of the dll

clr.AddReference('System.DirectoryServices.dll')

import sys

# Import the LDAP / ActiveDirectory interface
from System.DirectoryServices import *

# A few predefined values to simplify LDAP querying, pilfered from various internet-ey type places.
BINARY_PROPS = ('objectguid', 'objectsid', 'msexchmailboxsecuritydescriptor', 'msexchmailboxguid')
ACCOUNT_QUERY = "(&amp;(objectCategory=user)(objectClass=person)(sAMAccountName=_ACCOUNTNAME_))"
PRINCIPAL_QUERY = "(&amp;(objectCategory=user)(objectClass=person)(userPrincipalName=_PRINCIPALID__))"
GROUP_QUERY = "(&amp;(objectCategory=group)(sAMAccountName=_GROUPNAME_))"
PARTIAL_LAST_QUERY = "(&amp;(objectCategory=user)(sAMAccountName=*)(objectClass=person)(sn=_LASTNAME_*))" 
PARTIAL_NAME_QUERY = "(&amp;(objectCategory=user)(objectClass=person)(sAMAccountName=*)(sn=_LASTNAME_*)(givenname=_FIRSTNAME_*))" 
PARTIAL_FIRST_QUERY = "(&amp;(objectCategory=user)(objectClass=person)(sAMAccountName=*)(givenname=_FIRSTNAME_*))" 



class DirectoryEntryHelper(object):
  """ Helper class for wrapping
  and simplifying AD search results.
  As much fun as dealing with Microsoft style
  object hierarchies can be...
  TODO: Is there an easy way to inject things into __dict__ for dir &amp; ipy tab completion?
  """

  def __init__(self, entry):
      if isinstance(entry, SearchResult):
          self.__dict__['_entry'] = entry.GetDirectoryEntry()
      elif isinstance(entry, DirectoryEntry):
          self.__dict__['_entry'] = entry
      else:
          raise TypeError, "Invalid type '%s', don't know how to proxy it" % type(entry)

  @property
  def properties(self):
      return self._entry.Properties

  @property
  def keys(self):
      """Returns a comprehended list
      of the *defined* Keys on the Properties object.
      Any valid keys w/o values won't get listed.
      """
      return [key for key in self.properties.Keys]

  def __getattr__(self, key):
      """Proxies the Properties objects in the DirectoryEntry
      to provide a simple getter.
      First checks if DirectoryEntry has an attribute (property)
      matching requested key, and passes that if so.
      Otherwise, fetches a matching property.  If you have
      a name collison, use the .properties property on this object
      to get a clean copy of the Properties object
      TODO: Better support for multi-value keys (only returns first idx right now]
      """
      if hasattr(self._entry, key):
          return getattr(self._entry, key)
      elif self._entry.Properties[key].Count:
          return self._entry.Properties[key][0]
      else:
          return None

  def __setattr__(self, key, value):
      """Proxies the Properties objects in the DirectoryEntry 
      to provide a simpler setter.
      First checks if DirectoryEntry has an attribute (property)
      matching requested key, and sets on that if so.
      Otherwise, sets upon a matching property.  
      If the property has an existing count > 0, sets Index 0  
      If not, it does an Add().
      The way the MS Classes work, Properties.Keys only returns keys that
      have a value count.  However, any *valid* Key has a silent value which
      can be initialized via Add().
      Invalid keys throw an error, so you can't be arbitrary.
      I use a list of keyspace found at: 

          http://www.dotnetactivedirectory.com/\
              Understanding_LDAP_Active_Directory_User_Object_Properties.html

      If you have a name collison, use the .properties property on this object
      to get a clean copy of the Properties object
      TODO: Better support for multi-value keys (only sets first idx right now]
      """
      if hasattr(self._entry, key):
          setattr(self._entry, key, value)
      else:
          if self._entry.Properties[key].Count > 0:
              if self._entry.Properties[key].Count > 1:
                  print >> sys.stderr, "[warning] Key '%s' contains multiple values which we don't properly support yet.  Using Index 0"
              self._entry.Properties[key][0] = value
          else:
              self._entry.Properties[key].Add(value)
          # commit early, commit often...
          self._entry.CommitChanges()

  def __repr__(self):
      """Print all pretty like on console.
      My biggest pet peeve working in .Net is that most objects don't have
      any kind of __str__/__repr__ value on them to give you quick information
      on their contents.  ARGH!
      """
      return "<DirectoryEntryHelper:{sn=%s,givenName=%s,mail=%s,sAMAccountName=%s}>" %\
          (self.sn, self.givenName, self.mail, self.sAMAccountName)

class ActiveDirectory(object):
  """ ActiveDirectory interface class.
  Tested on IronPython 2.6RC + Mono 2.4.2.3 on Linux. YMMV.
  """

  # Active Directory Handle
  adh = None

  server = None
  domain = None
  baseDN = None

  def __init__(self, server, domain, username, password, baseDN=""):
      """Constructor to instantiate the LDAP connection.
      server is a resolvable address to your domain controller (e.g. the resolvable hostname or IP)
      domain should be the ActiveDirectory domain as specified by your admin.
      If you don't pass BaseDN, your 'domain' argument will be split on
      . and each piece will be passed as DC=<piece>. This will be used as your BaseDN
      E.G. domain aurigasv.local becomes "DC=aurigasv,DC=local"

      *** Any exception in connecting will be wrapped and rethrown, aborting construction.
      """
      try:
          # Setup the baseDN
          self.server = server
          self.domain = domain
          if baseDN:
              self.baseDN = baseDN
          else:
              for tier in domain.split('.'):
                  baseDN += "DC=%s," % tier
              self.baseDN = baseDN[:-1]
              #print "Established a parsed BaseDN of '%s'" % self.baseDN

          # Establish a directory object, with auth info, pointing at the authenticating user
          self.adh = DirectoryEntry("LDAP://%s/%s" % (server, self.baseDN))
          self.adh.AuthenticationType = AuthenticationTypes.Secure
          self.adh.Username = username
          self.adh.Password = password

          # connect and fetch a copy of the authenticating user
          filter = "sAMAccountName=%s" % username
          dsSystem = DirectorySearcher(self.adh, filter)
          dsSystem.SearchScope = SearchScope.Subtree
          result = dsSystem.FindOne()

          if not result:
              raise Exception, "Found no result searching for specified authentication user %s." % username

      except Exception, e:
          raise e, "Authentication failure.  Did you provide valid connection information / authentication credentials?"

  def get_helper(self, result):
      if hasattr(result, "__iter__"):
          return (DirectoryEntryHelper(item) for item in result)
      else:
          return DirectoryEntryHelper(result)


class ActiveDirectorySearcher(ActiveDirectory):

  def _search(self, filter, multi=True):
      """Semi-private method for searching.
      Note that "filter" needs to be a string which is a 
      *VALID* LDAP query/filter.  You should probably use one of the useful
      utility find methods instead of querying this directly
      unless you understand LDAP queries
      By default searches for one only, pass multi=True for multiple results.
      """
      print "[debug] Searching activedirectory for %s.  Allow multiple results? %s" % (filter, multi)
      searcher = DirectorySearcher(self.adh)

      searcher.ReferralChasing = ReferralChasingOption.All
      searcher.SearchScope = SearchScope.Subtree
      searcher.Filter = filter

      results = searcher.FindAll()

      if results.Count == 0:
          raise Exception, "No results found for search."

      if multi:
          # Generator for performance
          return (entry for entry in results)
      else:
          if results.Count > 1:
              raise Exception, "Found multiple results in single-search mode.  To support multiples, please pass multi=True to the find method."
          return results[0]  


  def find_last_name(self, last_name, multi=False, nowrap=False):
      """Searches by sn (lastname).
      This can be a partial query, as long as the first
      few letters of last_name are defined 
      By default searches for one only, pass multi=True for multiple results.
      If you want  raw .Net SearchResult(s) object(s), pass nowrap=True, otherwise
      you get it back wrapped in a helper class.
      """
      result = self._search(PARTIAL_LAST_QUERY.replace('_LASTNAME_', last_name), multi)
      if not nowrap:
          result = self.get_helper(result)

      return result

  def find_name(self, last_name=None, first_name=None, multi=False, nowrap=False):
      """Searches by sn (lastname) &amp; givenName( firstname) 
      This can be a partial query, as long as the first
      few letters of last_name are defined
      If firstname is None, it quietly passes into find_last_name.
      By default searches for one only, pass multi=True for multiple results.
      If you want  raw .Net SearchResult(s) object(s), pass nowrap=True, otherwise
      you get it back wrapped in a helper class.
      """
      if not last_name:
          raise Exception, "You must, at a minimum, specify last_name for a name search."

      if not first_name:
          result = self.find_last_name(last_name, multi)
      else:
          result = self._search(PARTIAL_NAME_QUERY.replace('_FIRSTNAME_', first_name).replace('_LASTNAME_', last_name), multi)
          if not nowrap:
              result = self.get_helper(result)

      return result

  def find_account(self, account_name, multi=False, nowrap=False):
      """Searches by  sAMAccountName (account_name), which should be unique
      This can be a partial query, as long as the first
      few letters of account_name are defined
      If firstname is None, it quietly passes into find_last_name.
      By default searches for one only, pass multi=True for multiple results.
      If you want  raw .Net SearchResult(s) object(s), pass nowrap=True, otherwise
      you get it back wrapped in a helper class.
      """
      result = self._search(ACCOUNT_QUERY.replace('_ACCOUNTNAME_', account_name), multi)
      if not nowrap:
          result = self.get_helper(result)

      return result

  def find_first_name(self, first_name, multi=False, nowrap=False):
      """Searches by givenName, which should be unique
      This can be a partial query, as long as the first
      few letters of account_name are defined
      If firstname is None, it quietly passes into find_last_name.
      By default searches for one only, pass multi=True for multiple results.
      If you want  raw .Net SearchResult(s) object(s), pass nowrap=True, otherwise
      you get it back wrapped in a helper class.
      """
      result = self._search(PARTIAL_FIRST_QUERY.replace('_FIRSTNAME_', first_name), multi)
      if not nowrap:
          result = self.get_helper(result)

      return result
{% endhighlight %}
