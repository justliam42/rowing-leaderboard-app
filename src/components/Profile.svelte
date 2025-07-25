<script>
  import { onMount } from 'svelte';
  import { currentUser, setCurrentUser, deleteSession } from '../utils/store.js';
  import { db } from '../services/database.js';
  import * as Card from '$lib/components/ui/card';   import * as Button from '$lib/components/ui/button';
  import * as Badge from '$lib/components/ui/badge';
  import { Separator } from '$lib/components/ui/separator';
  import UserSelect from './UserSelect.svelte';
  import CreateProfile from './CreateProfile.svelte';
  import ProfileCard from './ProfileCard.svelte';

  let user = null;
  let sessions = [];
  let showCreateProfile = false;
  
  // Delete workout state
  let deletingSessionId = null;
  
  // Success message state
  let successMessage = '';
  let showSuccess = false;

  currentUser.subscribe(value => {
    user = value;
    if (user) loadUserSessions();
  });

  async function loadUserSessions() {
    if (!user) return;
    try {
      sessions = await db.getSessions(user.id);
      sessions.sort((a, b) => {
        // Handle Firestore Timestamp objects properly
        const dateA = a.date?.toDate ? a.date.toDate() : new Date(a.date);
        const dateB = b.date?.toDate ? b.date.toDate() : new Date(b.date);
        return dateB - dateA;
      });
    } catch (error) {
      console.error('Failed to load sessions:', error);
    }
  }

  function formatDate(date) {
    // Handle Firestore Timestamp objects
    const jsDate = date?.toDate ? date.toDate() : new Date(date);
    return jsDate.toLocaleDateString('en-US', {
      month: 'short',
      day: 'numeric',
      year: 'numeric'
    });
  }

  function formatMeters(meters) {
    return meters.toLocaleString();
  }

  function handleUserSelected(selectedUser) {
    // User was selected, component will handle the rest
  }

  function handleUserCreated(newUser) {
    // User was created and set as current, hide create form
    showCreateProfile = false;
  }

  function handleCreateCancel() {
    showCreateProfile = false;
  }

  function showCreateForm() {
    showCreateProfile = true;
  }

  async function handleDeleteSession(sessionId) {
    if (confirm('Are you sure you want to delete this session? This action cannot be undone.')) {
      try {
        deletingSessionId = sessionId;
        await deleteSession(sessionId);
        
        // Update local state immediately for smoother UX
        sessions = sessions.filter(session => session.id !== sessionId);
        
        // Show success notification
        showSuccessMessage('Session deleted successfully');
      } catch (error) {
        console.error('Failed to delete session:', error);
        alert('Failed to delete session. Please try again.');
      } finally {
        deletingSessionId = null;
      }
    }
  }
  
  function showSuccessMessage(message) {
    successMessage = message;
    showSuccess = true;
    setTimeout(() => {
      showSuccess = false;
    }, 3000);
  }
</script>

<div class="container mx-auto p-4 max-w-2xl">
  <!-- Success notification -->
  {#if showSuccess}
    <div class="fixed top-4 right-4 z-50 bg-green-500 text-white px-4 py-2 rounded-lg shadow-lg flex items-center gap-2 animate-in slide-in-from-right duration-300">
      <span class="text-sm">✅</span>
      <span class="text-sm font-medium">{successMessage}</span>
    </div>
  {/if}

  {#if user}
    <div class="space-y-6">
      <ProfileCard 
        {user} 
        variant="profile" 
      />

      <Card.Card>
        <Card.Header>
          <Card.Title>Recent Sessions</Card.Title>
        </Card.Header>
        <Card.Content>
          {#if sessions.length === 0}
            <div class="text-center py-8">
              <p class="text-muted-foreground">No sessions logged yet. Start rowing!</p>
            </div>
          {:else}
            <div class="space-y-3">
              {#each sessions.slice(0, 10) as session, index}
                <div class="flex items-center justify-between p-3 rounded-lg border bg-card hover:bg-accent/50 transition-colors">
                  <div class="flex items-center gap-3">
                    <Badge.Badge variant="outline">
                      #{index + 1}
                    </Badge.Badge>
                    <span class="font-medium">{formatDate(session.date)}</span>
                  </div>
                  <div class="flex items-center gap-3">
                    <span class="font-semibold text-primary">{formatMeters(session.meters)}m</span>
                    <Button.Button 
                      size="sm" 
                      variant="ghost" 
                      onclick={() => handleDeleteSession(session.id)}
                      disabled={deletingSessionId === session.id}
                      class="text-destructive hover:text-destructive hover:bg-destructive/10 h-8 w-8 p-0"
                    >
                      {#if deletingSessionId === session.id}
                        <span class="animate-spin text-xs">⏳</span>
                      {:else}
                        🗑️
                      {/if}
                    </Button.Button>
                  </div>
                </div>
                {#if index < Math.min(sessions.length, 10) - 1}
                  <Separator />
                {/if}
              {/each}
            </div>
          {/if}
        </Card.Content>
      </Card.Card>
    </div>  {:else}
    <div class="space-y-6">
      <Card.Card>
        <Card.Header class="text-center">
          <div class="text-6xl mb-4">🚣</div>
          <Card.Title class="text-3xl">Welcome to MSOE Rowing!</Card.Title>
          <Card.Description class="text-lg">
            Select your profile or create a new one to get started.
          </Card.Description>
          <div class="mt-2">
            <Badge.Badge variant="secondary" class="text-sm font-semibold">
              DAY BY DAY!
            </Badge.Badge>
          </div>
        </Card.Header>
      </Card.Card>

      <UserSelect onUserSelected={handleUserSelected} />

      {#if !showCreateProfile}
        <div class="text-center">
          <Button.Button onclick={showCreateForm}>
            Create New Profile
          </Button.Button>
        </div>
      {/if}

      {#if showCreateProfile}
        <CreateProfile 
          onUserCreated={handleUserCreated}
          onCancel={handleCreateCancel}
        />
      {/if}
    </div>
  {/if}
</div>
